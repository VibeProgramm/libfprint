# Rebase recipe — GF511 fork on current upstream

Goal (not upstream-merge): track upstream master with minimum own delta.

## Branch layout (VibeProgramm/libfprint)

- `modern-1.94.100` — working branch. Base: freedesktop master at rebase
  time. Everything below lives here.
- Topic order inside (keep for cheap future rebases):
  1. build port (`meson*`, `tests/meson.build`, hwdb/udev lists)
  2. driver sources (`libfprint/drivers/goodixtls/`, `tests/goodixtls511/`,
     `tests/getrandom-seed.c`)
  3. driver fixes (read-loop error-spin, verify-drop, emulation API)
  4. warnings/style follow-ups

## How to rebase onto a new upstream master

1. `git fetch freedesktop master`
2. New worktree: `git worktree add -b modern-<ver> ~/src/wt-next freedesktop/master`
3. Port in this order (known conflict zones):
   - `meson.build` → `drivers_info` dict entry + gnutls helper block
   - `libfprint/meson.build` → `files()` entries
   - `tests/meson.build` → `drivers_tests` dict entry + `LD_PRELOAD`
     shim wiring for goodixtls511 only
   - `data/autosuspend.hwdb`, `libfprint/fprint-list-udev-hwdb.c` →
     keep 5110 out of the GENERATED unsupported block; if upstream
     added a `sync-*` target, run it instead of hand-editing
   - driver sources: plain copy, then re-apply topic (3) commits
     (`git cherry-pick -n` from the previous modern branch)
4. `meson setup /tmp/...` (no `-Dc_args` crutch — tree must be
   warning-free) + `ninja` + `meson test` (expect: unit OK, replay
   SKIP without umockdev, metainfo may fail offline — pre-existing).
5. Hardware gate (27c6:5110): install, restart fprintd, B1 (disconnect
   under active verify → journal delta ≤ 6, fast fail, daemon alive)
   + B2 (disconnect idle → delta ≤ 6, same PID), recovery without
   daemon restart.
6. Push branch, point AUR packaging (`PKGBUILD` `_branch`, workflow
   `FORK_BRANCH`) at it — `sync-pkgver` picks up the new pkgver.

## Deliberate differences from upstream (keep)

- GnuTLS>=3.6.0 TLS transport (upstream has no 5110 support at all).
- `getrandom-seed` LD_PRELOAD shim for the replay test (upstream's
  `test-emulation` lib covers only the emulation flag, not RNG).
- Custom SIGFM matcher (64x80 sensor can't do NBIS minutiae).
- No `verify` vfunc (uses upstream verify-via-identify fallback).
