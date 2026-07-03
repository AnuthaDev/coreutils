# uucore: drop the process-lifetime ARGV cache; derive util name / execution phrase from a single argv read

## Summary

`uucore` kept a permanent copy of all command-line arguments in a `static ARGV: LazyLock<Vec<OsString>>`, used to lazily derive `UTIL_NAME` and `EXECUTION_PHRASE` and to back `uucore::args_os()`. This PR removes that cache. Each binary entry point now reads argv exactly once, passes it to `uumain` as before, and explicitly initializes the utility name and execution phrase from that same read via two new functions:

- `uucore::init_util_name(arg)` — stores the utility name, stripping any directory path (`mkdir`, not `./target/debug/mkdir`)
- `uucore::init_execution_phrase(arg)` — stores the phrase shown in usage output (`sleep` or `coreutils sleep`)

`util_name()` and `execution_phrase()` keep working without initialization: they fall back to deriving the value from the process arguments on first use, replicating the previous behavior exactly (basename stripping, the `manpage` skip for `uudoc`, and the two-word `coreutils <util>` join when the utility is the second argument, gated on `get_utility_is_second_arg()`). The fallback only fires for callers that don't go through one of the wired entry points — in-process callers such as the fuzz targets, unit tests, and downstream crates that use `uucore` as a library.

## Rationale

The cache bought nothing and cost memory:

- `args_os()` is called once per process: `bin_inner!` passes it into `uumain`, and the multicall `coreutils` main reads it once to dispatch. Caching a once-used value for the process lifetime is pure overhead.
- The OS/libc keeps the original argv bytes alive for the whole process anyway. The static `Vec<OsString>` was a permanent duplicate — one heap allocation per argument plus the vector itself, up to ARG_MAX-sized invocations (e.g. `rm` fed by `xargs`).
- The old flow actually copied argv **twice**: once into the static, then again element-by-element (`.cloned()`) when handing the iterator to clap. The new flow makes a single transient copy that is consumed by argument parsing and freed.

Note on the mechanics (this corrects the reasoning in an earlier iteration of this branch): on Unix, `std` does *not* clone argv at startup — it only saves the raw `argc`/`argv` pointers. Every `std::env::args_os()` call clones all of argv into fresh `OsString`s. That is why the entry points initialize the name/phrase from the one read they already perform instead of reading argv a second time.

## Changes

- **`src/uucore/src/lib/lib.rs`**
  - Removed `static ARGV` and the `LazyLock` statics for `UTIL_NAME` / `EXECUTION_PHRASE`; replaced with `OnceLock`s.
  - Added `init_util_name()` / `init_execution_phrase()`. Both are first-write-wins (later calls are ignored), so harnesses that invoke `uumain` repeatedly in one process — the fuzz targets — are safe.
  - `util_name()` / `execution_phrase()` use `get_or_init` with a fallback that reproduces the old lazy derivation for un-wired callers.
  - `bin_inner!` (every standalone binary's `main`) peeks argv[0] from its single `args_os()` read and initializes both values before calling `uumain`. Deriving from argv[0] preserves the existing symlink/rename behavior (a binary copied to `nap` still reports `nap:` in errors).
  - `args_os()` / `args_os_filtered()` are now thin wrappers over `std::env::args_os()` (`wild::args_os()` on Windows); doc comments updated to say each call copies argv.
- **`src/bin/coreutils.rs`** (multicall): initializes both values in the dispatch arm — from the raw binary path when matched via argv[0] (symlink case), or from `<binary> <util>` when the utility is the second argument, preserving usage strings like `Try './coreutils ls --help'` byte-for-byte.
- **`src/bin/uudoc.rs`**: `manpage` generation initializes the execution phrase in `main` (where raw argv is available) and the utility name in `gen_manpage` after clap validates the utility, replacing the old `manpage`-skip hack for the wired path.

`set_utility_is_second_arg()` and its getter are retained: the multicall binary and `uudoc` still set the flag, and the lazy fallback derivation depends on it for un-wired callers.

## Breaking changes

No API removals — the new `init_*` functions are additive, and `util_name()` / `execution_phrase()` / `args_os()` keep their signatures. Two semantic changes for downstream consumers of the `uucore` crate:

1. **`uucore::args_os()` is no longer cheap to call repeatedly.** It previously returned clones from a process-lifetime cache; it now copies all of argv (and re-expands globs via `wild` on Windows) on every call. Call it once and reuse the result. In-tree callers were already single-call.
2. **`util_name()` / `execution_phrase()` can now be pinned by the embedding binary.** If a downstream binary calls the new `init_*` functions, those values win over derivation from argv. Callers that never init see identical behavior to before (same derivation, now computed on first use without the persistent argv copy).

## Testing

- `cargo test -p uucore --lib` (84 passed) and integration tests for `ls`, `mkdir`, `echo`, `test`, `sleep` (392 passed); full-workspace `cargo check` and clippy clean.
- Manual parity checks: multicall error/usage output (`sleep: missing operand` / `Try './target/debug/coreutils sleep --help'`), standalone binaries, a renamed standalone binary (argv[0] basename still used), `--version`, unknown-utility handling, and `uudoc manpage` / `uudoc completion` output.
