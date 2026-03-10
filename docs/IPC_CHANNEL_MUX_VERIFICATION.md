# IPC Channel Mux Migration: Pre-migration Verification

Date: 2026-03-10
Branch: `ipc-channel-mux-spike`
Purpose: Establish baseline before implementing the migration described in [IPC_CHANNEL_MUX_PLAN.md](IPC_CHANNEL_MUX_PLAN.md).

---

## Environment notes

- `mach` is functional after installing Python dependencies:
  ```
  pip install -r python/requirements.txt --break-system-packages
  pip install -r tests/wpt/tests/tools/wptrunner/requirements.txt --break-system-packages --ignore-installed packaging
  ```
- WPT tests could not be run: `mach test-wpt` requires a display server (Xvfb / X11) which is not available in this devcontainer.

---

## 1. Unit tests (`cargo test --workspace`)

**Result: PASS** (with one pre-existing failure)

All crates in the workspace compile and their unit tests pass, with one exception:

| Test | Package | Failure |
|---|---|---|
| `test_clear_cache` | `libservo --test network_manager` | Panics: "Already initialized: Opts" |

This failure is a **pre-existing test isolation bug**: two tests in the same binary both call `servo_config::opts::set_opts()`, which panics on the second call because the global is already initialized. It is unrelated to IPC channels.

---

## 2. Multiprocess integration test (`cargo test -p libservo --test multiprocess`)

**Result: PASS (exit code 0)**

The `tests/multiprocess.rs` custom harness compiled and ran successfully.

---

## 3. Servo with `--multiprocess` flag

```
target/debug/servo --multiprocess --headless -x about:blank
```

**Result: PASS (exit code 0)**

Servo starts, spawns a content process via `--content-process <socket>`, loads `about:blank` over IPC, and shuts down cleanly. The only warnings during the run are benign shutdown-race broken-pipe messages as threads tear down.

---

## 4. Baseline file descriptor counts

Measured while servo was running with `--multiprocess --headless about:blank`, approximately 3 seconds after launch:

| Process | FD count |
|---|---|
| Main (compositor/constellation) process | **67** |
| Content process (`--content-process`) | **40** |

These are the numbers to compare against after migration. The expectation is that under load (many tabs / fetches), FD count growth will be reduced because `SubSender<T>` does not consume additional OS file descriptors when sent over a mux channel.

---

## 5. `RUST_LOG=debug` — IPC error check

```
RUST_LOG=debug target/debug/servo --multiprocess --headless -x about:blank
```

**Result: No IPC errors.**

Debug-level logs show normal constellation/pipeline/script activity. No `ERROR` or unexpected `WARN` entries relating to IPC channels. (No mux-related logs are expected at this stage since `ipc-channel-mux` is not yet a dependency.)

---

## Pre-existing compiler warnings (not introduced by this work)

The following warnings exist in the codebase before migration begins:

- `background_hang_monitor`: `stack_ptrs`, `count` fields never read; `process_register` never used
- `script`: several unused imports (`NavigatorMethods`, `PerformanceMethods`, `Cell`, `DomRefCell`, `ImageKey`, `LayoutDom`)
- `libservo`: unused import `GenericSender` in `servo.rs`

---

## Summary

The codebase is in a clean baseline state. All unit tests pass (modulo the pre-existing `test_clear_cache` isolation bug). Servo runs correctly in multiprocess mode. The baseline FD counts (67 main / 40 content) are recorded for post-migration comparison.
