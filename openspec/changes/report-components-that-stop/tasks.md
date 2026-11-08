# Tasks

## 1. Both failing cases, before the fix

The defect was found twice by accident. Reproducing both on purpose is what makes the fix verifiable, and the second one is free: the lock can be made to give up on demand.

- [x] 1.1 Reproduce the lock case: make the locker unstartable, let the lock unit reach its start limit, and record that the diagnostic reports the component as running while the journal reports it as given up on
- [x] 1.2 Reproduce the agent case, or record why it cannot be reproduced on demand. It crashed on its own and the crash is in apport; killing it is not the same event, and the difference matters because `ExecStopPost` behaves differently for a signal than for a crash
- [x] 1.3 Establish whether `ExecStopPost` runs when a unit's main process is killed with `SIGKILL`, measured rather than assumed from the manual, because the whole approach rests on it

## 2. The two components that nothing follows

- [x] 2.1 Give the polkit agent's unit a Gridline-named drop-in with an `ExecStopPost`, the same shape already used for the three desktop shell components. Verify the effective `ExecStopPost` from systemd rather than from the file
- [x] 2.2 Give the lock the same, deciding between the unit's `ExecStopPost` and the entry point that already sets `LockedHint` at both ends. Record which and why: the entry point knows more, the unit is more systemd-shaped, and only one of them can be the place
- [x] 2.3 Verify the build gate that checks a drop-in starts the component its unit names still passes, and extend it if a drop-in that only records does not fit what that gate expects
- [x] 2.4 Verify neither drop-in changes what its unit does in a session that is not Gridline's, which is the rule every other drop-in here follows

## 3. The diagnostic stops claiming more than it knows

- [x] 3.1 Make the report distinguish what was observed from what is assumed. The record already carries `at=` and the diagnostic ignores it
- [x] 3.2 Verify against a record deliberately left stale, by writing one and confirming the report does not present it as current
- [x] 3.3 Verify the healthy case still reads as healthy and does not drown in qualifications. A diagnostic that hedges everything is as useless as one that is confidently wrong

## 4. The cases that produced this

- [x] 4.1 Verify the lock case: a lock that gives up is no longer reported as running, measured in the same way it was measured wrong
- [x] 4.2 Verify the agent case: an agent that dies mid-session is no longer reported as running
- [x] 4.3 Verify a component that stops cleanly and comes back reads as running again, so that the fix does not leave the record stuck at the first stop

## 5. Coherence

- [x] 5.1 Correct `docs/recovery.md`, whose worked example of the diagnostic shows five components running, now that one of those lines was observed under a mechanism that could not have known otherwise
- [x] 5.2 Verify each scenario of the modified requirement is exercised or recorded as unexercised with a reason, counting scenarios from the delta file rather than from the proposal's description of them

## Scenario accounting

Eight scenarios in the modified requirement, counted from
`specs/session-manageability/spec.md` and not from the proposal's description
of them. Six were inherited from `add-session-health-check` and two are new.
All eight were exercised on the LXD VM.

1. **A missing component is nameable** - both halves. The locker moved aside
   reads `not installed` with the path it was looked for at; a locker present
   and exiting 9 reads `failed` with `exit-code exited/9`. The first of those
   failed on the first attempt and is what found the `absent` precedence rule.
2. **A healthy session says so** - `Every component this session expects is
   accounted for`, exit 0. The wallpaper's record had to be set aside to reach
   it, because that component genuinely fails on this VM.
3. **The report distinguishes what refuses a session from what degrades it** -
   `polkit-agent ... (a session will not start without it)` beside
   `wallpaper ... (the desktop works without it, with less)`, and still correct
   after a stop, which is what the severity carry-forward is for.
4. **The diagnostic reads a record the session wrote** - the diagnostic names no
   unit another package owns. Its only two `systemctl` calls read Gridline's own
   targets.
5. **The diagnostic works when the session does not** - two halves. With no
   records at all, measured on the machine. With records present and the
   session's target inactive, measured on a copy of the program pointed at an
   absent target, because the live session could not be stopped to ask. The
   branch is exercised; **a machine actually in that state is not.**
6. **Reporting a gap never refuses a session** - the recorder replaced with one
   that always exits 1, on both ends of a blocking component's unit. Unit
   active, `Result=success`, one process.
7. **A component that stops after the session started is no longer reported as
   running** - the agent, stopped and restarted twice, reading `stopped` then
   `running` each time. The lock, given up on after three failed starts, reading
   `not installed`; and the real unlock during this work, which recorded `ready`
   through the `finished` verb without being arranged.
8. **The report says what it knows and not more** - records written on purpose
   that could not say when: one from the previous day, one with no time at all,
   one stamped in 2099. The first version of the report presented two of the
   three as ordinary current lines.

## Workflow follow-up

- Run `/opsx:verify` once both failing cases have been reproduced and then shown fixed.
- This change touches the lock unit that `recover-from-a-dead-screen-lock` introduces, so it lands after it rather than beside it.
