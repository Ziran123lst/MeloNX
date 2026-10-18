# exlaunch-dbg — MeloNX exlaunch mod instrumentation

This fork of **MeloNX 2.7** adds a diagnostic patch for running **exlaunch** mods
(模组 exefs 里的 `subsdk9` + `main.npdm`，无 `main.ips`).

## Background

- The loading pipeline for exlaunch mods is already present in MeloNX
  (`ModLoader` exefs overlay, NSO module list including `subsdk9`, rtld relocation).
- With an exlaunch mod installed (e.g. for 歌姬计划 MEGA39's), the game crashes
  **before entering the title** even though the same mod runs fine on PC Ryujinx.
- Crash report analysis (crash.txt) showed the exlaunch **abort fingerprint**
  (`0x69696969` in `x27`/`x13`, `x28 == 0`) — i.e. the loader hit an assertion
  (`EXL_ASSERT` / `R_ABORT_UNLESS`) during boot init and deliberately
  data-aborted (`src/common/exlaunch/diag/abort.cpp`).

## What the patch does

`patches/exlaunch_dbg.patch` only **adds logging** — zero behaviour change.
It instruments the syscalls exlaunch's loader depends on during boot:

- `svcQueryMemory`
- `svcMapProcessMemory` / `svcUnmapProcessMemory` (failures **and** successes)
- `svcGetInfo` (region lookups, `MesosphereCurrentProcess`)

Every failure is logged as `[exlaunch-dbg] <svc> failed (...)`. A
`MapProcessMemory OK` line before the crash means the abort happened at a
non-svc assertion (e.g. the `memcmp(ro, rw)` RW-alias check in
`RwPages`), while a `failed` line names the exact failing step.

## How to use

1. Build the IPA (see `.github/workflows/build-ipa.yml` — produces an unsigned
   IPA, sideload the same way you install MeloNX normally).
2. Install the exlaunch mod, launch the game, let it crash.
3. Share the console log lines starting with `[exlaunch-dbg]` (plus the last
   few lines before the crash) to pinpoint the failing assertion.

## Files

- `patches/exlaunch_dbg.patch` — unified diff against MeloNX 2.7
  (`src/Ryujinx.HLE/HOS/Kernel/SupervisorCall/Syscall.cs`)
- `.github/workflows/build-ipa.yml` — CI: applies the patch, builds the IPA
  (macOS 26 / Xcode 26, unsigned)
