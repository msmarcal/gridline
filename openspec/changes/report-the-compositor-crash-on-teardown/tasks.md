# Tasks

## 1. Which path, and what it costs

Before testing hypotheses, read what already happened. Twenty-five crashes are twenty-five recorded teardowns, and the journal around each one says which path ran.

- [ ] 1.1 For each of the 25 crashes, read the journal in the minute before it and classify the teardown: compositor exit on its own (the normal path, `wait` then stop), or Gridline stopping the target first (`stop_session`, the error path). Record the count of each. Verify by showing the line that identifies the path for at least one of each kind found
- [ ] 1.2 Measure what a crashing logout costs the next login: compare the first lines of the journal and `/var/crash` after a logout that crashed against one that did not, if one can be produced. Record the difference, or that there is none observable
- [ ] 1.3 Attempt a backtrace from the apport core, first through `debuginfod`, then through the `-dbgsym` package for `libaquamarine`. Record the top frames, or precisely why neither symbolised it

## 2. Eliminating hypotheses in order of cost

- [ ] 2.1 If a machine with working 3D is available, start and end one Gridline session on it and read the journal. Record whether the signature appears. If no such machine is available, record that, and that the remaining tasks separate Gridline from not-Gridline but cannot separate this VM from hardware
- [ ] 2.2 On the VM, start a session through the compositor's own entry rather than Gridline's - the diverted `hyprland.desktop`, reachable by undoing the diversion for the test - end it with the compositor's own exit binding, and read the journal. Verify the diversion is restored afterwards by `dpkg-divert --list`
- [ ] 2.3 Provoke Gridline's `stop_session` error path by withholding a required capability, and read whether the crash that follows carries the same signature. Verify the withheld capability is restored afterwards and a normal session starts

## 3. Writing it down

- [ ] 3.1 Add to the compositor's row in `docs/component-ownership.md` what its exit does on a machine without 3D, in the terms the investigation produced rather than in the terms this proposal guessed. Verify by reading the row back
- [ ] 3.2 Add a short section to `docs/recovery.md` for the person who finds an apport report after logging out, saying what it is, whether it cost them anything, and where it is tracked. Verify it is reachable from the document's own structure rather than appended at the end
- [ ] 3.3 If the cause is upstream, write the report in a form that could be filed - signature, versions, machine, backtrace if obtained, reproduction - and record where it would go. If the cause is Gridline's, open the change that fixes it and name it here. Verify by the artefact existing

## 4. Coherence

- [ ] 4.1 Update the proposal's "What Changes" to say what was delivered rather than what was planned, and update `design.md`'s Open Questions with what the journal said about the teardown path. Verify no artefact still describes the investigation in future tense
- [ ] 4.2 Verify every live document that mentions the crash agrees with the finding: `grep` for `aquamarine` and `segfault` across `docs/` and `openspec/changes/archive/`, and correct any that recorded the earlier guess

## Workflow follow-up

- Run `/opsx:verify` once group 2 has eliminated at least one hypothesis and group 3 has written the result where a user reads it.
- If task 2.3 shows Gridline's ordering contributes, the requirement about teardown belongs to a new change, as the proposal says: the change that looked is not the change that promises.
