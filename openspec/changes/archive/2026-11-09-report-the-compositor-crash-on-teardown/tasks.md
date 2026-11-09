# Tasks

## 1. Which path, and what it costs

Before testing hypotheses, read what already happened. Twenty-five crashes are twenty-five recorded teardowns, and the journal around each one says which path ran.

- [x] 1.1 For each of the 25 crashes, read the journal in the minute before it and classify the teardown: compositor exit on its own (the normal path, `wait` then stop), or Gridline stopping the target first (`stop_session`, the error path). Record the count of each. Verify by showing the line that identifies the path for at least one of each kind found
- [x] 1.2 Measure what a crashing logout costs the next login: compare the first lines of the journal and `/var/crash` after a logout that crashed against one that did not, if one can be produced. Record the difference, or that there is none observable
- [x] 1.3 Attempt a backtrace from the apport core, first through `debuginfod`, then through the `-dbgsym` package for `libaquamarine`. Record the top frames, or precisely why neither symbolised it

Results, from the VM's journal and its apport core (scripts were run on the VM; the instrument was corrected twice, see below):

- **1.1** 42 crashes, not 25, across six boots; the proposal's 25 was the current boot only (`journalctl -k` is one boot, `_TRANSPORT=kernel` is all of them). One signature in all 42: `segfault at 10 error 4 in libaquamarine.so.0.10.0`. Ordered against the line `Stopped target gridline-session.target` with microsecond timestamps: **29** crashed before the target was stopped, so the compositor was already gone when the launcher's `wait` returned (for example 2026-11-06T10:13:42.867 crash, stop 2.2 s later); **2** had Gridline stopping the target first (2026-10-29T16:51:05 and 16:51:17, both during the ordering-cycle defect the launcher's component check was written for, and both with the launcher's `required component missing` line, which is what identifies `stop_session`); **11** happened in login sessions that never reached Gridline's target at all, four of them under the archive's own wrapper (`start-hyprland` lines, "Hyprland exit cleanly" printed after the segfault). Attributed to sessions by logind's `New session` line, not by a time window. 31 crashes in the 35 times the target was reached; 3 sessions that reached it had no crash.
  - *Instrument errors found on the way.* The first classifier looked for a removal line within a 90 s window that ended 5 s after the crash and reported four crashes with no session removal; the removal came 6 to 7 s later, so those four were the window. And the first count of 25 used the wrong journal selector, as above.
- **1.2** Display manager came back in every case: a greeter within 40 s after 38 of the 42, and after the other four (the 2026-10-27 and 2026-10-29 sessions) the next login session began within 18 s to 2 min. The errors in the 40 s after a crash (`card0 not found`, `Failed to create wl_display`, `DRM_IOCTL_MODE_CREATE_DUMB`) are the components losing their compositor and the VM's missing 3D; they were not compared with a crash-free teardown, so nothing is attributed to the crash from them. On disk: one `Hyprland` report per user, 12 MB and 1.4 MB, overwritten and not stacked. Cost observed: nothing beyond the report.
- **1.3** Symbolised through `debuginfod.ubuntu.com` (gdb with `-iex "set debuginfod enabled on"`; the first attempt with the setting after the core loaded fell back to the question and answered no, which still named frames from the exported symbols). `#0 CLogger::log (this=0x0, AQ_LOG_DEBUG, "drm: Not disconnecting connector Virtual-1 because it's already disconnected")` at `DRM.cpp:1498`, from `SDRMConnector::disconnect`, `~SDRMConnector` (`DRM.cpp:1230`), `~CDRMBackend`, `~CBackend`, `__cxa_finalize`. The faulting instruction is `cmpq $0x0,0x10(%rdi)` with `rdi` zero. There is no `-dbgsym` package in the archive for it.
- **Found while searching for the trace:** it is a known upstream bug. [hyprwm/Hyprland discussion 12467](https://github.com/hyprwm/Hyprland/discussions/12467) carries the same trace on an Arch system with `eDP-1` and aquamarine 0.10.0, and records it fixed by `hyprwm/aquamarine@a43bedc`. Read directly rather than from a search summary: the commit is "backend: move logger to backend", 2025-12-02, two commits after `v0.10.0` and contained in `v0.11.0` onward (GitHub compare). Ubuntu 26.04's `libaquamarine9` is 0.10.0-2build1.

## 2. Eliminating hypotheses in order of cost

- [x] 2.1 If a machine with working 3D is available, start and end one Gridline session on it and read the journal. Record whether the signature appears. If no such machine is available, record that, and that the remaining tasks separate Gridline from not-Gridline but cannot separate this VM from hardware
- [x] 2.2 On the VM, start a session through the compositor's own entry rather than Gridline's - the diverted `hyprland.desktop`, reachable by undoing the diversion for the test - end it with the compositor's own exit binding, and read the journal. Verify the diversion is restored afterwards by `dpkg-divert --list`
- [x] 2.3 Provoke Gridline's `stop_session` error path by withholding a required capability, and read whether the crash that follows carries the same signature. Verify the withheld capability is restored afterwards and a normal session starts

Results:

- **2.1** No machine with working 3D was available to this project, and none was needed to answer the question: the same trace, on a real laptop panel, is in the upstream discussion above. That is another person's report and not a measurement of this project, and it is recorded as that. What it settles is that the VM is not the cause.
- **2.2** Not run as written. The four sessions in the journal that ran under the archive's own `start-hyprland` entry (logind sessions 12, 21, 132 and 135) are the measurement the task asks for, with the same signature and with none of Gridline's code involved; starting a fifth would need the diversion undone and a login at the display manager by a person, and could not add to the answer. The diversion was not touched: `dpkg-divert --list` still shows both Hyprland entries diverted by `gridline-session`.
- **2.3** Not provoked. The `stop_session` path ran twice on its own (2026-10-29 16:51, above) and the crash that followed has the same signature and the same backtrace location. Withholding a capability now would reproduce a path the project already has two recordings of.

## 3. Writing it down

- [x] 3.1 Add to the compositor's row in `docs/component-ownership.md` what its exit does on a machine without 3D, in the terms the investigation produced rather than in the terms this proposal guessed. Verify by reading the row back
- [x] 3.2 Add a short section to `docs/recovery.md` for the person who finds an apport report after logging out, saying what it is, whether it cost them anything, and where it is tracked. Verify it is reachable from the document's own structure rather than appended at the end
- [x] 3.3 If the cause is upstream, write the report in a form that could be filed - signature, versions, machine, backtrace if obtained, reproduction - and record where it would go. If the cause is Gridline's, open the change that fixes it and name it here. Verify by the artefact existing

Results:

- **3.1** The compositor row of `docs/component-ownership.md` says it, with the fixed version.
- **3.2** `docs/recovery.md` has a section, "A crash report appears after you log out", reached from a line before the recovery path as well as from its own heading.
- **3.3** `report.md` in this change, in a form that can be filed, with where it would go: Launchpad against `aquamarine`, asking for the cherry-pick. The cause is upstream, so no Gridline change follows from it.

## 4. Coherence

- [x] 4.1 Update the proposal's "What Changes" to say what was delivered rather than what was planned, and update `design.md`'s Open Questions with what the journal said about the teardown path. Verify no artefact still describes the investigation in future tense
- [x] 4.2 Verify every live document that mentions the crash agrees with the finding: `grep` for `aquamarine` and `segfault` across `docs/` and `openspec/changes/archive/`, and correct any that recorded the earlier guess

Results:

- **4.1** The proposal's "Why" and "What Changes" now give what was found, and the design's open questions are answered.
- **4.2** `grep` for `aquamarine` and `segfault` across `docs/` and `openspec/changes/archive/` found one live document with the wrong process (`docs/compositor-adapter.md`, which said `gdm-wayland-session`), now rewritten. Three archived task files carry the earlier guess (`add-gridline-configuration-defaults` 6.6, `add-application-launcher`, `add-desktop-shell-components`); they are dated records of what was believed then and were left as written, which is also what keeps the dead end visible.

## Workflow follow-up

- Run `/opsx:verify` once group 2 has eliminated at least one hypothesis and group 3 has written the result where a user reads it.
- If task 2.3 shows Gridline's ordering contributes, the requirement about teardown belongs to a new change, as the proposal says: the change that looked is not the change that promises.
