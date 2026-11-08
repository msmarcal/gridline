# Proposal

## Why

The screen locker died in a running Gridline session on the verification VM, and the person using that machine could not get back into it. The session stayed locked, which is correct, and there was nothing on screen to type a password into, which is not. Recovery took a second machine.

What the user saw was the compositor's own fallback, because `ext_session_lock_v1` keeps a session locked when the locking client dies. That screen tells them to run `hyprlock`, a program Gridline does not ship: Gridline declares `swaylock`. So the only instructions available at the moment of the failure point at something that is not installed.

The project already anticipated the locker dying and covered the half that is about safety. `platform-integration` requires that the session manager not claim a lock no component is maintaining and that the session stay locked, and both held: `LockedHint` went false while the compositor kept the session locked, which is the direction that leaves the session protected. **Nothing requires that the session stay recoverable by the person who owns it**, and that is the half that failed.

## What Changes

**This change ships implementation.**

- Requires that a Gridline session whose locker dies gets a locker back, so that a user at the machine can authenticate rather than needing another one. What that means in practice is the design's to settle: a locker the session restarts, or a lock entry point that does not exit when its locker does.
- Records the manual path in `docs/recovery.md`, which covers a session that does not start and says nothing about a session that starts and then locks the user out. The path has to name `swaylock`, because the screen the user is looking at names something else.
- Leaves the safety half alone. `gridline-lock` clearing `LockedHint` on every exit path is correct and the reasoning in its comment is correct; what the comment does not anticipate is that the compositor keeps the lock, which makes the clearing right for a different reason than the one written there.

**Out of scope.** Why the locker died. That is unknown, nothing recorded it, and this change is about surviving it rather than preventing it: a locker that cannot die is not a thing anyone can build, and a session that cannot be recovered when one does is a defect whatever the cause. The investigation is a task here and its result does not gate the rest.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

- `platform-integration`: one requirement is modified. "The session is lockable and the lock survives suspend" already carries what happens to the system's view when the lock component dies. It says nothing about the user, and a requirement that covers the machine's opinion of a locked session but not the person in front of it is covering the smaller half.

## Impact

- `packages/gridline/src/session/gridline-lock`: the lock entry point, which today runs one locker and exits when it does.
- `packages/gridline/src/session/gridline-idle.service`: where the idle timeout calls that entry point, if the answer turns out to be a unit rather than the script.
- `docs/recovery.md`: a section for a session that locks and will not unlock, naming the locker Gridline actually ships.
- `docs/component-ownership.md`: the screen locker row, whose fallback cell describes a session that refuses to start and not a session that strands its user.
- Verification runs on the LXD VM, where the failure happened once already and where a locker can be killed deliberately to make it happen again.
