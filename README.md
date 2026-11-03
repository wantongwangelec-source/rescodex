# rescodex

macOS local watchdog: when a Codex task stalls with `usage_limit_exceeded`, wait until live quota recovers, then automatically continue the same task with `codex exec resume`. No GUI, no notifications, zero third-party dependencies (Python standard library only). Covers both CLI and Codex Desktop sessions (they share `~/.codex/sessions/`).

Inspired by [joejoeha/codex-quota-resume](https://github.com/joejoeha/codex-quota-resume), with the GUI, tray, follow-up dialogs, and updater removed so only the core loop remains.

## How it works

Every 60 seconds, launchd starts `watcher.py`:

1. **Discover** — Scan `~/.codex/sessions/**/*.jsonl`. Track each session's open turn. A turn counts as quota-stalled when `task_complete` and `error.codex_error_info == "usage_limit_exceeded"`. Normal completion, cancel (`turn_aborted`), or a turn already resumed manually do not count. Thread id prefers `session_meta` in the session file (compatible with Desktop 0.163+ `rollout-<time>-<thread>_<file-id>.jsonl` dual-UUID names; filename parsing is fallback only). Parse results are cached by `(mtime, size)`; unchanged files are not reread.
2. **Confirm quota** — Call `account/rateLimits/read` via `codex app-server --stdio` and check `ordinaryUsageAllowed`. If that experimental API is unavailable, fall back to `rate_limits.*.resets_at` from the log plus a 120-second buffer.
3. **Resume** — Run `codex exec resume --skip-git-repo-check --json <session-UUID> "<resume prompt>"` in the original session working directory. Replay the interrupted turn's recorded `model`, `approval_policy`, and `sandbox` (mode plus workspace-write network / writable roots from `turn_context`) via `-c` so the resumed turn keeps the session settings. Legacy enum values the current CLI rejects (for example `untrusted`) are not overridden; config.toml is used instead and a log line is written. If the desktop app owns the session (active writer), fall back to `codex enqueue` so the desktop runtime handles it.

Dedup and crash safety:

- A file lock (`flock`) ensures only one monitor instance runs at a time.
- `state.json` keys are `session|turn`. Persist **before** dispatch; on failure, roll back and back off 300 seconds.
- Before dispatch, `pgrep` checks whether a codex process already owns that session (do not steal a manual takeover).
- If the session stays quiet for 10 minutes after dispatch and no live process remains → roll back and retry; enqueue mode gives up and does not re-enqueue.
- Only quota interruptions that happen **after monitoring started** are handled. Historical tasks are not revived (`monitoringSince` records first-run time).

## Install / uninstall

```bash
./install.sh     # self-test → write and load LaunchAgent (every 60s)
./uninstall.sh   # stop and remove; keep local state
```

Day-to-day:

```bash
python3 watcher.py --status    # status + current candidates
tail ~/Library/Application\ Support/codex-quota-resume/watcher.log
tail ~/Library/Application\ Support/codex-quota-resume/launchd.err.log  # crash stacks; normally empty
python3 watcher.py --dry-run   # report only; do not send
python3 watcher.py --self-test # built-in fixtures
python3 -m unittest test_watcher
```

When debugging, check three places: `watcher.log` records each decision (state changes, dispatch / failure / recovery; a heartbeat every 10 minutes during long waits); `state.json` uses `lastCheckedAt` as the per-minute heartbeat and `sent` as send history; `launchd.err.log` only has content when the process exits abnormally.

## Boundaries

- Monitor queries do not consume quota; real resumes consume Codex quota normally.
- Does not read or upload login credentials, redeem reset coupons, or buy quota. `model`, `approval_policy`, and `sandbox` are replayed from the interrupted session's `turn_context` as recorded; values that cannot be replayed fall back to config.toml and are logged.
- The resume prompt is a fixed "continue the original task" message and does not expand the original task's authorization.
- `app-server` is experimental; Codex upgrades may change behavior. If it breaks, the watcher only falls back to `resets_at` (a few minutes later).
- Automated tests use temporary-directory fixtures only and never send messages to real tasks. On 2026-09-18 a full real "quota exhausted → reset" cycle was verified: within 1 minute after reset, quota recovery was detected and the task auto-resumed; the turn finished normally after about 9 minutes with no manual intervention.
- The machine must be awake, logged into Codex, and online. Checks do not run during macOS sleep; the next wake cycle catches up.

## Files

| File | Role |
|---|---|
| `watcher.py` | All core logic (single file, standard library) |
| `test_watcher.py` | State-machine fixture tests |
| `install.sh` / `uninstall.sh` | LaunchAgent install / uninstall |

MIT License.
