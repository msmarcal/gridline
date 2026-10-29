# Tasks

## 1. The selection on record

- [ ] 1.1 Record the selection of `swaylock`, `swayidle` and `hyprpolkitagent` in `docs/component-ownership.md`, replacing the rows that name candidates, and verify neither row still says the component is not selected
- [ ] 1.2 Remove both entries from the deferred-declarations table in the same document, and verify the table no longer lists a component this change declares, since a deferral that has happened is not a deferral
- [ ] 1.3 Record for each selection what decided it, by the measurement rather than by preference: which Hyprland libraries each candidate links, which pull a desktop environment, and which speak the session lock protocol. Verify the figures in the document match the packages by re-reading the archive metadata rather than copying from design.md
- [ ] 1.4 Correct the passage in `docs/security-and-manageability.md` that analyses the candidate locker's PAM stack, since it names a component this change does not select. Read the selected locker's own PAM file and record what it contains, and verify the document no longer presents the unselected one as the project's locker
- [ ] 1.5 Verify no live document still names `hyprlock` or `hypridle` as the project's choice, by grep, and that any surviving mention is clearly a rejected candidate with the reason

## 2. The components are declared

- [ ] 2.1 Declare `swaylock`, `swayidle` and `hyprpolkitagent` as `Depends` of `gridline-session`, and verify with `dpkg-deb -f` on the built package rather than by reading the control file
- [ ] 2.2 State the `universe` support gap for the three in the `gridline-session` long description, using the sentence the live documents already carry, and verify the wording is identical by comparing against `docs/component-ownership.md`
- [ ] 2.3 Build and verify both gates pass and the ownership gate's file count is unchanged, since this change ships no new file into a package
- [ ] 2.4 Install on the baseline VM and verify the three resolve from the archive alone, recording how many packages apt fetched

## 3. The launcher gates on components

- [ ] 3.1 Add the blocking-component check to the launcher, in the phase that runs before anything starts, and verify by grep that it names no compositor so the isolation gate still reports zero references outside the adapter path
- [ ] 3.2 Verify an absent blocking component ends the launch with that component named, using a stub installation with one of the three removed, and read the failure from `journalctl -t gridline-session` rather than from the exit status
- [ ] 3.3 Verify the check runs before the compositor, by confirming no compositor process was created when a component is absent
- [ ] 3.4 Verify a reported capability does not substitute for a present component: with the adapter reporting `lock-signaling` and the locker removed, the session must still refuse, and the failure must name the locker rather than the capability
- [ ] 3.5 Verify the check does not consult systemd unit state, since a unit that restarts on failure would otherwise read as evidence the component is present. Confirm by inspection that the check tests for the component itself

## 4. The gate's reference list

- [ ] 4.1 Correct `ext_session_lock_v1` to `ext_session_lock_manager_v1` in the isolation check's reference list. The adapter was corrected when a live registry disproved the first name; the gate's copy was not, so it has been scanning for a string nothing can contain. Verify by planting the correct name outside the adapter path and confirming the build fails, then removing it and confirming the build passes
- [ ] 4.2 Verify every other protocol name in the gate's list appears in the adapter, by comparing the two in both directions and recording any name in one and not the other, including none

## 5. Verification in a session

Execution note: this group needs the session running on the VM, started from the greeter. Two of its checks cannot be performed on that machine and say so rather than being skipped silently.

- [ ] 5.1 Start a Gridline session with all three components installed and verify it reaches a desktop, so the new gate is shown not to refuse a correct installation
- [ ] 5.2 Lock the session manually and verify the contents are no longer accessible and that authentication is required to return, recording which key or command was used since nothing is bound yet
- [ ] 5.3 Suspend the locked session and resume it, and verify it is still locked and still requires authentication. The machine supports suspend: `/sys/power/state` offers `freeze mem` and `sleep.target` exists
- [ ] 5.4 Verify suspend and resume return a working session with display, input and network restored, and record anything that did not come back
- [ ] 5.5 Trigger a privileged action and verify the prompt appears and accepts credentials, recording which action was used
- [ ] 5.6 Record that battery observability was not exercised and why: the machine has no battery, confirmed by zero real `upower` devices and no `BAT*` under `/sys/class/power_supply/`. Name what would exercise it
- [ ] 5.7 Record that lid handling was not exercised and why: the machine has no lid, confirmed by the absence of `/proc/acpi/button/lid` and by `logind` reporting a default rather than an observation. Name what would exercise it

## 6. Coherence

- [ ] 6.1 Verify the requirement added by this change is satisfied by something a reader can check, and name for each of its scenarios the file or measurement that satisfies it
- [ ] 6.2 Verify the nine existing scenarios this change implements are each either exercised or recorded as unexercised with a reason, and that none is left unaccounted for
- [ ] 6.3 Verify the five blocking components the architecture names are now either declared by a Gridline package or recorded as owned by a change that will declare them, with none left unattributed

## Workflow follow-up

- Run `/opsx:verify` once the session has locked and resumed on the VM.
- Archive the change, then propose `add-gridline-configuration-defaults`, which binds keys to the components this change selects and is the first change to make the session look like Gridline.
