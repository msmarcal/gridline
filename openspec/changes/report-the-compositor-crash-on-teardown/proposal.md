# Proposal

## Why

The compositor segfaults when a Gridline session ends. Measured on the verification VM: 25 crashes, one signature, against 27 sessions that reached the session target. Essentially every session this project has run has ended with its compositor crashing, and nothing in the project says so.

It predates every change made so far - the first is dated 2026-11-06, before the desktop shell existed - so it is not a consequence of anything Gridline built. It was found while verifying something else, and recorded in an archived change as a finding nobody owned.

## What Changes

**This change ships investigation and documentation. Whether it ships a fix depends on what the investigation finds, and the tasks say so rather than promising one.**

- Establishes whether the crash is specific to this machine's lack of 3D acceleration or happens on hardware, because the two have different owners and only one of them is Gridline's.
- Establishes whether Gridline's teardown sequence contributes. The session stops a target, which stops components, while the compositor is also exiting; whether the crash is the compositor's own exit path or a consequence of that ordering is unknown and decidable.
- Records what a user loses to it, which is the part that decides urgency. A crash during teardown may cost nothing but a report on disk, or it may cost the session's last state.
- Says so in the project's documentation either way, so that the next person to read a crash report after logging out does not start looking in Gridline.

**Out of scope.** Fixing the compositor. Gridline integrates with Hyprland through an adapter and does not patch it; if the cause is upstream, this change produces a report good enough to send there, not a patch.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

None, and `skip_specs: true` is set to say so rather than leave the change looking unfinished.

The requirement this might one day add - that a session's teardown leave no crash behind - cannot be written honestly before knowing whether Gridline causes the crash. Writing it now would mean the project requiring of itself something it has no evidence it controls, which is the kind of requirement that gets quietly reinterpreted later rather than met.

So this change measures and writes down. If the investigation shows Gridline's ordering contributes, the requirement belongs to the change that fixes it: the change that looked is not the change that promises.

## Impact

- `docs/component-ownership.md`: the compositor's row, which describes what its absence costs and says nothing about what its exit does.
- `docs/recovery.md`: a user who finds a crash report after logging out has nowhere to read that this is known.
- `packages/gridline/src/session/gridline-session`: its teardown, if the ordering turns out to contribute.
- Verification runs on the LXD VM, which is where the 25 crashes are, and needs a second reading somewhere with working 3D to separate the two explanations.

## Prior measurement

Recorded because the numbers are the reason this change exists rather than a hunch about instability.

Twenty-five segfaults between 2026-11-06T09:49 and 2026-11-08T20:48, every one of them `segfault at 10 ... error 4 in libaquamarine.so.0.10.0`. Normalising the addresses first suggested two distinct signatures; that was the normalisation leaving a residue, and there is one.

All twenty-five fall within sixty seconds of a `Removed session` line. Against twenty-seven `Reached target` lines for the Gridline session over the same period, which is why this reads as every session rather than an occasional fault.

`aquamarine` is the compositor's backend library, and the same library is what failed on a GitHub runner with `CBackend::create() failed!` when `add-ci-vm-smoke-tests` asked whether a compositor could start there. The two failures are at opposite ends of a session's life and in the same place, which is a reason to suspect the machine's graphics rather than the project, and not yet evidence of it.
