# Tasks

## 1. The selection on record

- [x] 1.1 Record the selection of `swaylock` and `hyprpolkitagent` in `docs/component-ownership.md`, replacing the rows that name candidates, and verify neither row still says the component is not selected. The idle daemon is deliberately not among them: counting the blocking components showed the project records six and that none is an idle daemon, so selecting one was scope this change added to itself. Its measurements stay as candidate evidence for `add-gridline-configuration-defaults`
- [x] 1.2 Remove both entries from the deferred-declarations table in the same document, and verify the table no longer lists a component this change declares, since a deferral that has happened is not a deferral
- [x] 1.3 Record for each selection what decided it, by the measurement rather than by preference: which Hyprland libraries each candidate links, which pull a desktop environment, and which speak the session lock protocol. Verify the figures in the document match the packages by re-reading the archive metadata rather than copying from design.md
- [x] 1.4 Correct the passage in `docs/security-and-manageability.md` that analyses the candidate locker's PAM stack, since it names a component this change does not select. Read the selected locker's own PAM file and record what it contains, and verify the document no longer presents the unselected one as the project's locker
- [x] 1.5 Verify no live document still names `hyprlock` or `hypridle` as the project's choice, by grep, and that any surviving mention is clearly a rejected candidate with the reason

## 2. The components are declared

- [x] 2.1 Declare `swaylock` and `hyprpolkitagent` as `Depends` of `gridline-session`, and verify with `dpkg-deb -f` on the built package rather than by reading the control file. `swayidle` was declared here and then removed; the removal was verified the same way, and on the VM by `apt-get autoremove` offering to remove it, which is the dependency being gone rather than the control file being edited
- [x] 2.2 State the `universe` support gap for both in the `gridline-session` long description, using the sentence the live documents already carry, and verify the wording is identical by comparing against `docs/component-ownership.md`
- [x] 2.3 Build and verify both gates pass and account for the ownership gate's file count. It moves from 10 to 11: this change ships one new file, the session's systemd user target, which was not true when this task was written
- [x] 2.4 Install on the baseline VM and verify both resolve from the archive alone, recording how many packages apt fetched

## 3. The launcher gates on components

- [x] 3.1 Add the blocking-component check to the launcher, in the phase that runs before anything starts, and verify by grep that it names no compositor so the isolation gate still reports zero references outside the adapter path
- [x] 3.2 Verify an absent blocking component ends the launch with that component named, using a stub installation with one of them removed, and read the failure from `journalctl -t gridline-session` rather than from the exit status
- [x] 3.3 Verify the check runs before the compositor, by confirming no compositor process was created when a component is absent
- [x] 3.4 Verify a reported capability does not substitute for a present component: with the adapter reporting `lock-signaling` and the locker removed, the session must still refuse, and the failure must name the locker rather than the capability
- [x] 3.5 Verify the check does not consult systemd unit state, since a unit that restarts on failure would otherwise read as evidence the component is present. Confirm by inspection that the check tests for the component itself

## 4. The gate's reference list

- [x] 4.1 Correct `ext_session_lock_v1` to `ext_session_lock_manager_v1` in the isolation check's reference list, and verify by planting each reference outside the adapter path one at a time and confirming the build fails for every one. Correcting the name exposed a second defect: the check scanned twice, printing in one pass and counting in the other, and with `set -e` a grep that does not match ended the counting loop's body non-zero, so the subshell exited at the first file holding no reference. The summary read zero while the lines above it named a leak, and the build passed. Counting and reporting now come from one pass
- [x] 4.2 Verify every other protocol name in the gate's list appears in the adapter, by comparing the two in both directions and recording any name in one and not the other, including none

## 5. Verification in a session

Execution note: this group needs the session running on the VM, started from the greeter. Two of its checks cannot be performed on that machine and say so rather than being skipped silently.

- [x] 5.1 Start a Gridline session with the blocking components installed and verify it reaches a desktop, so the new gate is shown not to refuse a correct installation. First exercised with three installed, before the idle daemon was dropped. Task 7.6 starts the session again with the dependency set this change actually ships, and the VM has had `swayidle` autoremoved so that run is not resting on a package the launcher no longer checks for
- [x] 5.2 Lock the session manually and verify the contents are no longer accessible and that authentication is required to return, recording which key or command was used. Used `swaylock` typed in a terminal, since nothing is bound to a key yet. The screen became inaccessible and showed light grey with no indicator, which is the locker's documented default: the background defaults to A3A3A3 and the indicator appears on input, which is why `--indicator-idle-visible` exists. Authentication was genuinely enforced rather than assumed, evidenced by a `pam_unix(swaylock:auth): authentication failure` in the journal before the successful unlock. One finding for `add-gridline-configuration-defaults`: `LockedHint` stayed `no` throughout, so the compositor was locked while logind did not know it, and wiring the idle daemon to logind's lock signal belongs to the change that configures them
- [x] 5.3 Record that the lock surviving suspend was not exercised and why: the machine cannot suspend. `systemd-sleep` reports `Failed to put system to sleep. System resumed again: Operation not supported`, and writing to `/sys/power/state` returns an I/O error, although the file lists `freeze mem`, which is what the kernel was built to support rather than what the hardware accepts. What the attempt did show is recorded rather than discarded: `systemd-sleep` froze and thawed `user.slice`, and the session came through it with the compositor running, the locker's two processes alive, and the screen still locked. Exercising the scenario needs hardware that suspends
- [x] 5.4 Record that suspend and resume returning a working session was not exercised, for the same reason as 5.3 and with the same evidence. A freeze and thaw of `user.slice` is not a suspend cycle, and claiming the scenario on it would be claiming the weaker thing as the stronger one
- [ ] 5.5 Trigger a privileged action and verify the prompt appears and accepts credentials, recording which action was used
- [ ] 5.6 Record that battery observability was not exercised and why: the machine has no battery, confirmed by zero real `upower` devices and no `BAT*` under `/sys/class/power_supply/`. Name what would exercise it
- [ ] 5.7 Record that lid handling was not exercised and why: the machine has no lid, confirmed by the absence of `/proc/acpi/button/lid` and by `logind` reporting a default rather than an observation. Name what would exercise it

## 6. Coherence

- [ ] 6.1 Verify the requirement added by this change is satisfied by something a reader can check, and name for each of its scenarios the file or measurement that satisfies it
- [ ] 6.2 Verify the nine existing scenarios this change implements are each either exercised or recorded as unexercised with a reason, and that none is left unaccounted for
- [ ] 6.3 Verify the blocking components the architecture names are now either declared by a Gridline package or recorded as owned by a change that will declare them, with none left unattributed. Count them from the fallback column rather than from any sentence that states a number: the page said five in two places and carried six rows

## 7. The components that run for the session get started

This group is last in the file and belongs after group 3 in the work. It was written once group 4 was complete and the session had been started on the VM, which is when declaring the polkit agent turned out to leave it installed and never running: the archive ships its unit disabled, `graphical-session.target` refuses a manual start, and the unit is conditioned on a `WAYLAND_DISPLAY` the user manager does not have. Renumbering the groups above would churn a file that has been committed in pieces, so the numbering records the order the work was found in rather than the order it is done in.

- [x] 7.1 Measure all three obstacles on the baseline rather than reasoning about them: that `dpkg -L` lists the unit and no symlink, that `graphical-session.target.wants` holds nothing of the agent's, that `graphical-session.target` carries `RefuseManualStart=yes`, and that the unit's condition is on `WAYLAND_DISPLAY`. Record each with the command that showed it
- [x] 7.2 Ship `gridline-session.target` in `gridline-session`, wanting the units the session needs and pulling in `graphical-session.target` as a dependency. Verify on the VM that `systemctl --user show -p Wants -p BindsTo` reports the two separately, since the launcher reads the first and must not see the second
- [x] 7.3 Install the target to `/usr/lib/systemd/user/` without enabling it and without a maintainer script, and verify `dpkg -L` lists the file and no symlink, so Gridline enables nothing under a target every graphical session shares
- [x] 7.4 Extend the launcher to name the Wayland socket to the user manager, start the target, and refuse the session when a unit the target wants is not running. Verify by grep that the launcher names no unit itself, since the target is the one declaration. The grep is not silent and should not be: the launcher names `/usr/libexec/hyprpolkitagent`, which is the binary the pre-flight check looks for, not a unit. What must appear nowhere in the launcher is `hyprpolkitagent.service`, and it does not
- [x] 7.5 Verify the isolation gate still reports zero references outside the adapter path, and record that it fired once on this work and on what, since a gate that has never failed is not known to work
- [ ] 7.6 Start a session on the VM and verify the target is active, that `graphical-session.target` came up with it, and that the agent is running, reading each from `systemctl --user is-active` rather than from the session appearing to work
- [ ] 7.7 Verify the session refuses to start when the agent is installed but its unit cannot run, and that the failure names the unit in `journalctl -t gridline-session`
- [ ] 7.8 Log out and verify nothing the target started is still running, since `BindsTo` is what makes the session leave no residue for the next one

## Workflow follow-up

- Run `/opsx:verify` once the session has locked and resumed on the VM.
- Archive the change, then propose `add-gridline-configuration-defaults`, which binds keys to the components this change selects and is the first change to make the session look like Gridline.
