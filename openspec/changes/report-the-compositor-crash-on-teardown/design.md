# Design

## Context

See proposal.md for the numbers. What the project already knows that narrows the question, because two of the three hypotheses are already weakened by things measured for other reasons.

**Gridline does not touch the compositor until it has exited, on the normal path.** The launcher's main sequence is `wait "$compositor_pid"` and then `systemctl --user stop gridline-session.target`: it waits for the compositor to leave on its own and tears down afterwards. The crash recorded at 03:36:57 on 2026-11-08 came from a logout chosen in the command menu, which asks the adapter, which runs the compositor's own exit dispatcher. Gridline was still in `wait`. Whatever crashed, crashed inside the compositor's own exit, before the launcher did anything.

There is a second path, `stop_session`, which stops the target first and the compositor second. It runs on the error paths - a missing capability, a failed component check - and it is the only place Gridline's ordering could plausibly matter. Whether any of the 25 crashes came through it is not known, and is the one thing about Gridline's own behaviour worth measuring.

**The same library fails at the other end of a session's life on a machine with no graphics.** `add-ci-vm-smoke-tests` asked whether a compositor could start on a hosted runner. It could not: `CBackend::create() failed!` in `aquamarine`, with a real DRM device present. The crash here is `segfault at 10 error 4 in libaquamarine.so.0.10.0`, at exit. Two failures in the backend library at the two moments it touches the hardware, on two machines that both lack a working GPU, is a pattern and not yet a cause.

**The VM has no 3D.** This is measured and documented elsewhere in the project: the wallpaper daemon fails with `DRM_IOCTL_MODE_CREATE_DUMB: Permission denied` because Mesa falls back to software rendering that needs DRM master. Every one of the 25 crashes happened on this machine.

**The core is not symbolised.** apport captured a 12 MB report and its `StacktraceTop` is empty, so the location inside `aquamarine` is unknown. A backtrace is what turns "in the backend library" into something a maintainer can act on.

## Goals / Non-Goals

**Goals:**

- Know which of three things is crashing: the machine's graphics, the compositor's exit path on any machine, or Gridline's teardown ordering. These have three different owners.
- Know what a user loses to it, which decides whether it matters beyond tidiness.
- Have it written where a user who finds a crash report after logging out would look.

**Non-Goals:**

- Patching the compositor or its backend. Gridline integrates through an adapter and does not carry patches; if the cause is upstream, the deliverable is a report good enough to file.
- Suppressing the crash report. Hiding a symptom to make a log cleaner is the opposite of what this project does with the things it finds.

## Decisions

### Three hypotheses, tested in the order that eliminates the most

**The machine's graphics.** The cheapest discriminator is a machine with working 3D. One session started and ended there, with the journal read afterwards, either reproduces the crash or does not. If it does not, the VM is the cause and the deliverable is a sentence in the documentation. This is first because it costs one login and may end the investigation.

**The compositor's own exit.** If the crash reproduces on hardware, the next question is whether it needs Gridline at all. The compositor's own session entry - `hyprland.desktop`, which the archive ships and Gridline diverts - starts it with none of Gridline's configuration, units or teardown. A session through that entry, ended with the compositor's own exit binding, that still crashes is a crash Gridline merely inherits.

**Gridline's ordering.** Only if the crash does not reproduce through the compositor's own entry does Gridline's teardown become the suspect. The specific test is the `stop_session` path: provoke it by withholding a required capability, and read whether that crash carries the same signature as the 25.

### A backtrace is worth getting whichever hypothesis wins

The archive carries debug symbols for `libaquamarine` through `debuginfod` or a `-dbgsym` package, and `gdb` on the core apport kept would give a stack. This is done regardless of the outcome above, because a report to anyone - upstream, Ubuntu, or this project's own future - is worth more with a frame in it than with a library name.

### What a user loses is measured, not assumed

A crash at exit may cost nothing: the session was ending, the process was going away, and the only artefact is a report on disk. Or it may cost the last thing the compositor does on its way out, which for a Wayland compositor can include restoring the display's state. The measurement is what the next login looks like after a crashing logout versus a clean one, and what accumulates under `/var/crash` across logins - the report found was 12 MB, and apport keeps one per executable so it is overwritten rather than stacked, which is why this has not filled a disk.

### Where it is written down

`docs/component-ownership.md` gets a sentence in the compositor's row about what its exit does on a machine without 3D, because that row is where somebody reading about the compositor looks. `docs/recovery.md` gets a short section, because a person who has just logged out and found an apport dialog on the next login is following the recovery document and not the ownership table.

## Risks / Trade-offs

- **The hardware reading needs hardware this project does not have.** → The design names it as the first discriminator because it is cheap *if available*. If it is not, the compositor-entry test on the VM still separates "Gridline" from "not Gridline", which is the half that decides what this project does.
- **Symbolising the core may need packages the VM does not have and a network it may not reach.** → `debuginfod` is the first attempt and `-dbgsym` the second; failing both, the core stays and the report says it is unsymbolised rather than pretending.
- **Finding the cause upstream means the project ships nothing but words.** → That is the honest deliverable for an upstream bug, and the words are what stops the next person spending an afternoon on it.

## Open Questions

- Whether any of the 25 came through the `stop_session` error path rather than the normal exit. The journal around each crash can say which path ran, and nobody has read it that way.
- Whether the crash leaves the display in a state the next session has to recover from. Nothing has been observed, and nothing has been looked for.
