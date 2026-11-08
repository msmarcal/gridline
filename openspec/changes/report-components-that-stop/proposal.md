# Proposal

## Why

`gridline doctor` reports a component as running when it is not. Measured twice in one session, on both of the components it is most important to be right about:

- The polkit agent crashed, with a crash report timestamped inside that session, while the diagnostic reported `polkit-agent  running`.
- The screen lock gave up after three failed starts, with the journal naming the reason, while the diagnostic reported `locker  running`.

The cause is the same for both and is structural rather than a slip. The record the diagnostic reads is written at two moments: the start program writes one for each desktop shell component, and an `ExecStopPost` in that component's Gridline drop-in updates it when the component stops. The blocking components have neither. The launcher writes `started` for them once, at session start, and nothing ever writes again, so for those two "running" means "was installed when the session began".

This is a defect in `add-session-health-check`, which shipped the diagnostic. It is reported here rather than patched there because that change is archived and because a diagnostic that lies about a blocking component is worth its own record.

## What Changes

**This change ships implementation.**

- Makes the record for the blocking components follow them, so that a component which stops or dies after the session started stops being reported as running.
- Decides what "running" may mean when nothing is watching. A record written once is honest only if the diagnostic says when it was written and stops claiming more than it knows.
- Verifies against both of the cases that produced this: a locker that gives up, and an agent that dies mid-session.

**Out of scope.** Why the polkit agent crashed, which is unknown and is a separate question from whether anything noticed. The platform integrations Gridline does not start, which `docs/component-ownership.md` already records as a different problem with no mechanism yet.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

- `session-manageability`: one requirement is modified. "A session component that is not running is reported" is the requirement this change fixes, and its scenario about naming a missing component is satisfied today only for the components that happen to have a Gridline drop-in. The requirement says nothing about *when* the report was true, which is the gap that let a write-once record satisfy it on paper.

## Impact

- `packages/gridline/src/session/gridline-session`: the launcher, which writes the blocking components' records once.
- `packages/gridline/src/session/`: whatever follows those components afterwards, which may be a drop-in of Gridline's on the agent's unit, the same shape already used for the shell components.
- `packages/gridline/src/tools/gridline`: the diagnostic, if a record's age becomes part of what it reports.
- `packages/gridline/src/session/gridline-lock.service`: the lock unit, which has no record of its own at all today.
- Verification runs on the LXD VM, where both failing cases have happened on their own and where the lock can be made to give up on demand.

## Prior measurement

The defect was found by a component crashing in front of us rather than by review: the polkit agent's crash raised an apport dialog in a Gridline session, and checking what the diagnostic said about it is what exposed the write-once record. The lock case was found minutes later while verifying a different change, which is what established that the cause is structural and not particular to one component.
