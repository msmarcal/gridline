# Proposal

## Why

The screen locker died in a running Gridline session on the verification VM, and the person using that machine could not get back into it. The session stayed locked, which is correct, and there was nothing on screen to type a password into, which is not. Recovery took a second machine.

What the user saw was the compositor's own fallback, because `ext_session_lock_v1` keeps a session locked when the locking client dies. That screen tells them to run `hyprlock`, a program Gridline does not ship: Gridline declares `swaylock`. So the only instructions available at the moment of the failure point at something that is not installed.

**The project already required this and the implementation did not deliver it.** `platform-integration` has carried a scenario since `add-gridline-configuration-defaults` saying that when the lock component dies while the session is correctly held locked, the user can restore a working lock screen and return to their session **from the machine itself, rather than the session being unrecoverable without ending it**. Recovery took a second machine, so that scenario was not satisfied by the shipped code.

The safety half of the same requirement did hold: `LockedHint` went false while the compositor kept the session locked, which is the direction that leaves the session protected.

**This proposal first said that nothing required recoverability, which was wrong**, and the error is recorded rather than quietly fixed because it changes what this change is. A gap in the specifications is something to design; a requirement the code did not meet is something that should have been caught, and was not, by anything the project runs.

## What Changes

**This change ships implementation.**

- Makes a Gridline session whose locker dies get a locker back, so that the scenario already in `platform-integration` is true of the shipped code rather than only of the specification. What that means in practice is the design's to settle: a locker the session restarts, or a lock entry point that does not exit when its locker does.
- Records the manual path in `docs/recovery.md`, which covers a session that does not start and says nothing about a session that starts and then locks the user out. The path has to name `swaylock`, because the screen the user is looking at names something else.
- Leaves the safety half alone. `gridline-lock` clearing `LockedHint` on every exit path is correct and the reasoning in its comment is correct; what the comment does not anticipate is that the compositor keeps the lock, which makes the clearing right for a different reason than the one written there.

**Out of scope.** Why the locker died. That is unknown, nothing recorded it, and this change is about surviving it rather than preventing it: a locker that cannot die is not a thing anyone can build, and a session that cannot be recovered when one does is a defect whatever the cause. The investigation is a task here and its result does not gate the rest.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

- `platform-integration`: one requirement is modified, and less than first thought. "The session is lockable and the lock survives suspend" already covers both halves of a dead lock component: the system's view of it, and the user being able to restore a lock screen from the machine itself. Neither needed writing. What it does not cover is that the instructions a user finds at that moment have to name the locker the session actually uses, which is the one scenario this change adds.

## Impact

- `packages/gridline/src/session/gridline-lock`: the lock entry point, which today runs one locker and exits when it does.
- `packages/gridline/src/session/gridline-idle.service`: where the idle timeout calls that entry point, if the answer turns out to be a unit rather than the script.
- `docs/recovery.md`: a section for a session that locks and will not unlock, naming the locker Gridline actually ships.
- `docs/component-ownership.md`: the screen locker row, whose fallback cell describes a session that refuses to start and not a session that strands its user.
- Verification runs on the LXD VM, where the failure happened once already and where a locker can be killed deliberately to make it happen again.
