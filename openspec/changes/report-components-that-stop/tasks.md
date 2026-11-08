# Tasks

## 1. Both failing cases, before the fix

The defect was found twice by accident. Reproducing both on purpose is what makes the fix verifiable, and the second one is free: the lock can be made to give up on demand.

- [ ] 1.1 Reproduce the lock case: make the locker unstartable, let the lock unit reach its start limit, and record that the diagnostic reports the component as running while the journal reports it as given up on
- [ ] 1.2 Reproduce the agent case, or record why it cannot be reproduced on demand. It crashed on its own and the crash is in apport; killing it is not the same event, and the difference matters because `ExecStopPost` behaves differently for a signal than for a crash
- [ ] 1.3 Establish whether `ExecStopPost` runs when a unit's main process is killed with `SIGKILL`, measured rather than assumed from the manual, because the whole approach rests on it

## 2. The two components that nothing follows

- [ ] 2.1 Give the polkit agent's unit a Gridline-named drop-in with an `ExecStopPost`, the same shape already used for the three desktop shell components. Verify the effective `ExecStopPost` from systemd rather than from the file
- [ ] 2.2 Give the lock the same, deciding between the unit's `ExecStopPost` and the entry point that already sets `LockedHint` at both ends. Record which and why: the entry point knows more, the unit is more systemd-shaped, and only one of them can be the place
- [ ] 2.3 Verify the build gate that checks a drop-in starts the component its unit names still passes, and extend it if a drop-in that only records does not fit what that gate expects
- [ ] 2.4 Verify neither drop-in changes what its unit does in a session that is not Gridline's, which is the rule every other drop-in here follows

## 3. The diagnostic stops claiming more than it knows

- [ ] 3.1 Make the report distinguish what was observed from what is assumed. The record already carries `at=` and the diagnostic ignores it
- [ ] 3.2 Verify against a record deliberately left stale, by writing one and confirming the report does not present it as current
- [ ] 3.3 Verify the healthy case still reads as healthy and does not drown in qualifications. A diagnostic that hedges everything is as useless as one that is confidently wrong

## 4. The cases that produced this

- [ ] 4.1 Verify the lock case: a lock that gives up is no longer reported as running, measured in the same way it was measured wrong
- [ ] 4.2 Verify the agent case: an agent that dies mid-session is no longer reported as running
- [ ] 4.3 Verify a component that stops cleanly and comes back reads as running again, so that the fix does not leave the record stuck at the first stop

## 5. Coherence

- [ ] 5.1 Correct `docs/recovery.md`, whose worked example of the diagnostic shows five components running, now that one of those lines was observed under a mechanism that could not have known otherwise
- [ ] 5.2 Verify each scenario of the modified requirement is exercised or recorded as unexercised with a reason, counting scenarios from the delta file rather than from the proposal's description of them

## Workflow follow-up

- Run `/opsx:verify` once both failing cases have been reproduced and then shown fixed.
- This change touches the lock unit that `recover-from-a-dead-screen-lock` introduces, so it lands after it rather than beside it.
