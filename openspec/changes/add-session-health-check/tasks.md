# Tasks

## 1. The measurement the design defers

This comes first because it decides the mechanism for half the change, and because the design names what would disqualify each candidate rather than assuming one works.

- [ ] 1.1 Establish what `Upholds=` does with a unit whose `Condition` fails: left alone, or started again on every cycle. Measure on the verification VM, which already has a component that cannot start, by giving that unit a condition that is false and counting starts over two minutes against the 59 already measured without one. A count near zero settles it; a count near 59 disqualifies the design's preferred candidate
- [ ] 1.2 If 1.1 disqualifies it, measure the next candidate before writing any of it, and record which candidate the design named and what disqualified it. If 1.1 settles it, record that too: the design predicted this one and a prediction confirmed is worth as much as one corrected
- [ ] 1.3 Establish where a record can live so that the diagnostic reads it when the session is not running, and so that it does not outlive the session that wrote it. Read what already exists rather than inventing: the session has a runtime directory, and the project has a `/var/lib/gridline` it created for a different purpose

## 2. The session records a gap

- [ ] 2.1 Record a component that is absent or fails to start, at the point the session already knows, which is the program that starts each component. Verify by reading the record after a session with a component genuinely missing, not one staged by stopping a unit
- [ ] 2.2 Record the Gridline name of the component rather than the unit name, and whether it is blocking. Verify by reading the record and confirming no unit name appears in it, since a reader asking why their bar is missing should not need to know which package provides it
- [ ] 2.3 Verify the record does not survive the session that wrote it, by logging out and back in with the component restored and confirming the record is gone rather than stale
- [ ] 2.4 Verify that recording changes no startup decision: the launcher's refusal behaviour is unchanged, read from its code and from a session started with a non-blocking component absent

## 3. The command

- [ ] 3.1 Ship a `gridline` command in `gridline-tools`, which ships no executable today, with `doctor` as its first subcommand. Verify with `dpkg -c` on the built package that the command belongs to the package `package-delivery` names as its owner
- [ ] 3.2 Report every component the session expects, naming which are running and which are absent or failed, from the record rather than by inspecting units other packages own. Verify by grep that the implementation names no archive unit
- [ ] 3.3 Report a healthy session as healthy, rather than printing nothing. Verify by running it on a session with every component present and confirming the output distinguishes that from the diagnostic having failed to run
- [ ] 3.4 Say for each reported component whether its absence refuses a session or degrades it, so severity is not inferred from the component's name. Verify against the blocking list the session target declares rather than against a list written in the command
- [ ] 3.5 Verify the command runs and reports when no session is running at all, which is the case it exists for. Run it from a text console with no graphical session and confirm it answers rather than erroring

## 4. Stopping the hold

- [ ] 4.1 Implement whichever mechanism group 1 established, so that a component that cannot start stops being started. Verify by counting starts over two minutes on the VM against the 59 measured before the change
- [ ] 4.2 Verify a component that stops after having worked is still restarted, which is the distinction the requirement draws and the thing most likely to be broken by whatever stops the other case. Kill a healthy component and confirm it returns
- [ ] 4.3 Verify the failure count belongs to the session: a component that failed enough to be given up on is tried again in the next session, rather than staying given up on across a logout
- [ ] 4.4 Verify giving up is reported, by confirming the component appears in the diagnostic after the session stopped holding it. A component that was quietly abandoned is worse than one that loops, because the loop is at least visible in the journal

## 5. Verification in a session

Execution note: this group needs the session running on the VM, started from the greeter, and it needs the wallpaper daemon left unmasked, since the machine's inability to run it is the genuine instance of the condition this change reports.

- [ ] 5.1 Verify the whole path on the real failing case: log in, let the wallpaper daemon fail, and confirm the session gives up on it, records it, and the diagnostic names it as a degrading gap
- [ ] 5.2 Verify the stock Ubuntu session is unchanged, by logging into it, since every change that touches session units has to answer this and two have now failed to
- [ ] 5.3 Verify the desktop still reaches a usable state with the gap present, which is the requirement the reporting must not quietly turn into a blocking check

## 6. Coherence

- [ ] 6.1 Remove the paragraph in `docs/component-ownership.md` that records the promise as unimplemented, and correct its count while removing it: it says seventeen cells across five live documents, and there are fourteen cells across three, with five further mentions in prose. Count both before and after rather than trusting either number
- [ ] 6.2 Verify every live cell promising a capability gap is now true, by listing them and naming for each what reports it. A cell that no longer applies is a cell to change, not one to leave
- [ ] 6.3 Record in `docs/recovery.md` how a user finds out what is missing, which today is by noticing. That document is where someone goes when the desktop is wrong, and it is the one place the command has to be named
- [ ] 6.4 Verify each scenario of the added and modified requirements is exercised or recorded as unexercised with a reason, counting scenarios from the delta files rather than from the proposal's description of them

## Workflow follow-up

- Run `/opsx:verify` once the diagnostic has been run from a session started at the greeter and from a console with no session.
- Archive, then the application launcher and the command menu, which `docs/user-experience.md` calls the element worth the most to the experience.
