# Report: Hyprland segfaults at exit in libaquamarine 0.10.0 (Ubuntu 26.04)

Written so that it can be filed as it stands. Not filed: that is the project's
owner's to do, and nothing here has been sent anywhere.

## Where it would go

- Ubuntu: a Launchpad bug against the `aquamarine` source package (binary
  `libaquamarine9`), asking for the upstream fix to be cherry-picked. This is the
  one that matters to Gridline's users, because the archive ships 0.10.0.
- Upstream needs nothing: the defect is already fixed, see below.

## Summary

`/usr/bin/Hyprland` segfaults while the process is exiting. The faulting call is
`Aquamarine::CLogger::log` with `this == nullptr`, reached from
`SDRMConnector::disconnect` while `CBackend` is destroyed by `__cxa_finalize`.
The compositor has already finished its work and the display manager carries on
normally, so the visible effect is a crash report on disk.

Upstream fixed it in `hyprwm/aquamarine` commit `a43bedc` ("backend: move logger
to backend", 2025-12-02). The commit is two commits after `v0.10.0` and is
contained in `v0.11.0` and every later tag. Ubuntu 26.04 ships 0.10.0.

## Versions

- `libaquamarine9` 0.10.0-2build1
- `hyprland` 0.53.3+ds-4
- Ubuntu 26.04 LTS (resolute), x86_64, kernel 7.0

## Signature

Kernel line, identical in all 42 occurrences on one machine:

    Hyprland[PID]: segfault at 10 ip ... error 4 in libaquamarine.so.0.10.0[85456,...]

Backtrace from the 12 MB apport core of 2026-11-08 03:36:57, symbols fetched
from `https://debuginfod.ubuntu.com`:

    #0  Aquamarine::CLogger::log (this=0x0, level=AQ_LOG_DEBUG,
          str="drm: Not disconnecting connector Virtual-1 because it's already disconnected")
    #1  Aquamarine::SDRMConnector::disconnect   DRM.cpp:1498
    #2  Aquamarine::SDRMConnector::~SDRMConnector   DRM.cpp:1230
    #3-9 shared-pointer release of SDRMConnector
    ... Aquamarine::CDRMBackend::~CDRMBackend, Aquamarine::CBackend::~CBackend
    #9  __cxa_finalize   (process exit)

The faulting instruction is `cmpq $0x0,0x10(%rdi)` with `rdi == 0`, which is
the `segfault at 10` of the kernel line.

## Machine

An LXD virtual machine with a virtio GPU and no 3D acceleration, output
`Virtual-1`. This is not what makes it happen: the same trace, with `eDP-1` as
the connector, is in
[hyprwm/Hyprland discussion 12467](https://github.com/hyprwm/Hyprland/discussions/12467),
from an Arch system with aquamarine 0.10.0 on real hardware, which was
resolved by `a43bedc`.

## Reproduction

End any Hyprland session whose DRM backend has a connector: the compositor's own
exit, a `SIGTERM`, or a display manager logout. It is not specific to Gridline;
see the tally in `tasks.md` 1.1.

## What was ruled out

- Gridline's session code: 11 of the 42 happened in sessions that never reached
  Gridline's target, four of them under the archive's own `start-hyprland`
  wrapper.
- The machine's missing 3D: reproduced on real hardware by others.
