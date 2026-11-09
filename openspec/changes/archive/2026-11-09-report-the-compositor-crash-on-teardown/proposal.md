# Proposal

## Why

The compositor segfaults when a Gridline session ends. Measured on the verification VM: 42 crashes across six boots, one signature, against 35 times a session reached the session target (the first count, 25 against 27, was one boot). Essentially every session this project has run has ended with its compositor crashing, and nothing in the project said so.

It predates every change made so far - the first is dated 2026-11-06, before the desktop shell existed - so it is not a consequence of anything Gridline built. It was found while verifying something else, and recorded in an archived change as a finding nobody owned.

## What Changes

**This change shipped investigation and documentation, and no fix: the cause is upstream.**

- Established that the crash is not specific to this machine's lack of 3D. The same backtrace, with a real laptop panel as the connector, is reported upstream against the same library version. That is another person's measurement, recorded as such.
- Established that Gridline's teardown does not contribute: 29 of 42 crashed before Gridline touched anything, 11 happened in sessions that never reached its target (four under the archive's own wrapper), and the two where Gridline stopped the session first have the same trace.
- Recorded what a user loses to it: a crash report on disk, nothing else observed.
- Took a symbolised backtrace, found it is a known bug in `libaquamarine` 0.10.0, fixed upstream in 0.11.0, and wrote a report that can be filed with Ubuntu (`report.md`).
- Said so in `docs/recovery.md`, `docs/component-ownership.md` and `docs/compositor-adapter.md`, which had named the wrong process.

**Out of scope.** Fixing the compositor. Gridline integrates with Hyprland through an adapter and does not patch it; the cause being upstream, this change produced a report good enough to file and not a patch.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

None, and `skip_specs: true` is set to say so rather than leave the change looking unfinished.

A requirement that a session's teardown leave no crash behind could not be written honestly before knowing whether Gridline causes the crash, and it does not: the investigation found it does not. No requirement is added, because the project would be requiring of itself something it does not control.

## Impact

- `docs/component-ownership.md`: the compositor's row, which describes what its absence costs and says nothing about what its exit does.
- `docs/recovery.md`: a user who finds a crash report after logging out has nowhere to read that this is known.
- `packages/gridline/src/session/gridline-session`: unchanged. Its teardown was found not to contribute.
- Verification runs on the LXD VM, which is where the 42 crashes are, and needs a second reading somewhere with working 3D to separate the two explanations.

## Prior measurement

Recorded because the numbers are the reason this change exists rather than a hunch about instability.

Twenty-five segfaults between 2026-11-06T09:49 and 2026-11-08T20:48, every one of them `segfault at 10 ... error 4 in libaquamarine.so.0.10.0`. Normalising the addresses first suggested two distinct signatures; that was the normalisation leaving a residue, and there is one.

All twenty-five fall within sixty seconds of a `Removed session` line. Against twenty-seven `Reached target` lines for the Gridline session over the same period, which is why this reads as every session rather than an occasional fault.

`aquamarine` is the compositor's backend library, and the same library is what failed on a GitHub runner with `CBackend::create() failed!` when `add-ci-vm-smoke-tests` asked whether a compositor could start there. The two failures are at opposite ends of a session's life and in the same place, which is a reason to suspect the machine's graphics rather than the project, and not yet evidence of it.
