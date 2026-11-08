# Tasks

## 1. The measurement the design defers

This comes first because it decides the mechanism for half the change, and because the design names what would disqualify each candidate rather than assuming one works.

- [x] 1.1 **It does not leave the unit alone, and the design's preferred candidate is disqualified as a complete answer.** The baseline was reconfirmed live before the test rather than taken from the record: 29 starts in 60 seconds, which is the 5-per-10-seconds rate limit the unit declares.

  With a false `ConditionPathExists`, over 120 seconds: **5 starts against 59, and 191 condition evaluations**, each one logged as `skipped, unmet condition`, at about 1.6 per second and still going when the window closed.

  So the condition does the half it was chosen for, which is to stop the component being launched, and fails the half that was decided: `Upholds=` keeps trying, so a crash loop becomes a logging loop. The journal is noisier than before and the CPU cost is lower, and neither is "stops being started again"
- [x] 1.2 **All four candidates the design listed are disqualified together, by the same fact.** Each tried to make `Upholds=` give up, and it does not: the manual says it restarts a unit found inactive **or** failed, continuously, and 1.1 measured that holding even when the unit is skipped rather than started.

  The mechanism chosen instead is to stop using `Upholds=` for these components, because `Restart=always` plus the start limit already mean what the requirement says: a component that has run and stops is restarted, and one that cannot start stops being started because nothing revives it after the limit. The three move under a Gridline-owned `gridline-shell.target` that the session target `Wants=`, so that the launcher's blocking check, which reads `Wants=`, cannot be made to refuse a session by one of them.

  **`Restart=always`, not `on-failure`, and that is a measurement and not a preference.** The notification daemon's shipped unit declares no `Restart=` at all; the other two declare `on-failure`. These components come back today when killed because `Upholds=` revives them, not because they restart themselves, so a clean kill would stop being recovered the moment it goes.

  The alternative weighed and declined was teaching the launcher to read an explicit list of blocking units instead of inferring it from `Wants=`. It reads better and rewrites a verified mechanism, and the indirection of one target costs less
- [x] 1.3 **`$XDG_RUNTIME_DIR/gridline/`, one file per component.** It is session-scoped by construction rather than by cleanup code, and it exists whenever the user has any session at all, including a text console opened after a graphical login failed, which is the case the requirement names. `/var/lib/gridline` was the other candidate and is wrong here: it is where the install records what it disabled, and a health record there would outlive the session that wrote it, which is exactly what the design says the failure count must not do.

  One file per component rather than one file with a line each, because three components can stop at the same moment and an append race is a defect nobody would reproduce on purpose.

  **A problem this task found rather than answered.** The wrapper knows a component is absent, because it checks before exec. It cannot know the component failed, because by then it has been replaced by it. systemd knows, and the Gridline-named drop-in already attached to each unit is where to ask: `ExecStopPost=` runs when the service stops and has `$SERVICE_RESULT` and `$EXIT_STATUS`. That keeps the recording inside a file Gridline owns rather than inspecting a unit it does not, which is the line the project context draws

## 2. The session records a gap

- [x] 2.1 Recorded by `gridline-component-state`, which one program owns so the format cannot drift between the three places that touch it. Two moments write it: the starting program, which knows a component is not there because it checks before replacing itself with it, and an `ExecStopPost` in the Gridline drop-in already attached to each unit, which is the only thing left that knows how a component ended.

  Verified against the genuinely impossible case rather than a staged one: the wallpaper daemon on the verification VM records `state=failed`, `detail=core-dump dumped/SEGV`. The record reads `bar started`, `notifications started`, `wallpaper failed` against real states of active, active and failed.

  **A defect found by the record disagreeing with the machine, and it was mine.** The record showed components as stopped while they were running. The recorder wrote its fields in a block ending with `[ -n "$detail" ] && printf`, and under `set -e` a false test makes the block fail, which sent the tolerant `|| exit 0` out before the move: written to the temporary file and never arriving. Every state carrying a detail was recorded and the one state without one, `started`, never was, which is why it looked like a race rather than a bug. Proven in isolation before fixing, and the comment at the site says so
- [x] 2.2 The record is keyed on the Gridline name: `bar`, `notifications`, `wallpaper`, never `waybar.service` and never the package. Verified by reading the record directory, where the file names are the roles and no unit name appears in any field.

  **Whether a component is blocking is deliberately not in the record.** It is not a property of what happened, it is a property of how the session is wired, and the session's own targets already declare it: `gridline-session.target` wants what refuses a session and `gridline-shell.target` wants what does not. Writing it into each record would be a second copy of a fact that already has a home, and the one that drifts
- [ ] 2.3 Verify the record does not survive the session that wrote it, by logging out and back in with the component restored and confirming the record is gone rather than stale
- [ ] 2.4 Verify that recording changes no startup decision: the launcher's refusal behaviour is unchanged, read from its code and from a session started with a non-blocking component absent

## 3. The command

- [x] 3.1 `/usr/bin/gridline` ships in `gridline-tools`, read back from the built package with `dpkg -c`: `-rwxr-xr-x ./usr/bin/gridline`, and it is the only executable in a package that until now contained a directory and the files debhelper puts there. `package-delivery` names that package as the owner of the command surface, which this is the first of.

  Exit codes are a command's quietest contract and were measured without a pipe, because `$?` after one reports the filter: 64 for every usage error, 1 when a component is not running, 0 when none is
- [x] 3.2 Reports from the record. Run against the live session, where the wallpaper daemon genuinely cannot start on this machine:

  ```
  bar              running
  notifications    running
  wallpaper        failed  (the desktop works without it, with less)
  ```

  Verified by grep that the command names **no** archive unit: zero occurrences each of `waybar`, `mako`, `hyprpaper`, `swaylock` and `hyprpolkitagent`. The only units it reads are `gridline-session.target` and `gridline-shell.target`, which are Gridline's own, and it reads them to report the wiring rather than to decide a component's state
- [x] 3.3 Three outcomes read differently, which is the point: every component running says so in a sentence, a component that is not says which and what it costs, and **nothing recorded says that and explains why**, rather than printing an empty list a reader would have to interpret.

  The empty case was exercised against an empty runtime directory and reads: no Gridline session has recorded anything for this user, which is what you see before a first login and after logging out of every session, because the record lives and dies with them. That is a fact about the machine rather than a failure of the command, and it exits 0.

  **Not exercised: the all-running case on this machine**, because the wallpaper daemon cannot start here. It needs a machine that renders, and is recorded as unexercised rather than counted
- [ ] 3.4 Say for each reported component whether its absence refuses a session or degrades it, so severity is not inferred from the component's name. Verify against the blocking list the session target declares rather than against a list written in the command
- [x] 3.5 Run with no session and no `XDG_RUNTIME_DIR` at all: it answers, in the same words as the empty-record case, and exits 0. It does not error, and it does not report a healthy session either, which are the two ways this could have been wrong

## 4. Stopping the hold

- [x] 4.1 **0 starts in two minutes, against 59.** The component that cannot start on the verification VM now reaches `failed` with `NRestarts=5`, the journal says `Start request repeated too quickly`, and nothing revives it. The three moved from `Upholds=` on the session target to `Wants=` on `gridline-shell.target`, which the session target wants; `systemctl --user show` reads back `Wants=hyprpolkitagent.service gridline-shell.target` and `Upholds=gridline-idle.service`, so the launcher's blocking check still sees one service and a target that is active whatever its own wants did
- [x] 4.2 Killed and returned, both of them, with Gridline's configuration on the command line rather than the bare binary. This was the thing most likely to break and it needed the drop-in to carry `Restart=always`: the notification daemon's shipped unit declares no `Restart=` at all, and both components came back before this change only because `Upholds=` revived them. A clean `SIGTERM` is not a failure, so `on-failure` would not have been enough either
- [ ] 4.3 Verify the failure count belongs to the session: a component that failed enough to be given up on is tried again in the next session, rather than staying given up on across a logout
- [x] 4.4 The component that the session stopped holding is the one the diagnostic names: after it reached `failed` with the start limit exhausted, `gridline doctor` reports `wallpaper  failed  (the desktop works without it, with less)` and exits 1.

  This is the pairing the change exists for. Giving up without reporting would have been a worse state than the loop it replaced, because a loop is at least visible in the journal, and a component silently abandoned is a desktop missing a piece with nothing anywhere saying so

## 5. Verification in a session

Execution note: this group needs the session running on the VM, started from the greeter, and it needs the wallpaper daemon left unmasked, since the machine's inability to run it is the genuine instance of the condition this change reports.

- [ ] 5.1 Verify the whole path on the real failing case: log in, let the wallpaper daemon fail, and confirm the session gives up on it, records it, and the diagnostic names it as a degrading gap
- [ ] 5.2 Verify the stock Ubuntu session is unchanged, by logging into it, since every change that touches session units has to answer this and two have now failed to
- [ ] 5.3 Verify the desktop still reaches a usable state with the gap present, which is the requirement the reporting must not quietly turn into a blocking check

## 6. Coherence

- [x] 6.1 **Narrowed rather than removed, because this change closes part of the promise and not the whole of it.** The task assumed removal; counting the cells said otherwise. Of the fourteen, **three** are components the session starts and now reports, the bar, the notification daemon and the wallpaper daemon, joined by the locker and the polkit agent. The other eight are platform integrations, NetworkManager, PipeWire, WirePlumber, BlueZ, UPower, XWayland, snapd and the two portal rows, which Gridline integrates with and does not start.

  Reporting those would mean a Gridline command inspecting the state of units other packages own, which is the coupling the design of this change refuses by name. So the paragraph now says which rows are true, which are owed, and why the remainder is a different problem rather than an oversight.

  The counts were taken again rather than trusted: **14 cells on three pages, 5 further mentions in prose, 3 scenarios in `openspec/specs/`**. The wrong number this change was created to correct, seventeen cells across five documents, is gone from the page
- [ ] 6.2 Verify every live cell promising a capability gap is now true, by listing them and naming for each what reports it. A cell that no longer applies is a cell to change, not one to leave
- [x] 6.3 The document already had a step promising `gridline doctor`, written before the command existed. It now describes what the command does: the components it names, that it answers without a session running, that nothing recorded reads differently from everything healthy, and its exit codes. It also says plainly what it does not cover, pointing at the page that says why.

  **The worked example in it is output that was observed, not output that was composed.** A first draft showed the locker and the polkit agent in the list; those lines have not been seen, because the session on the verification machine started before the launcher that records them. The example is now the three lines that were measured, with a sentence for the two that appear on a session started afterwards, and that sentence is what the next login verifies
- [ ] 6.4 Verify each scenario of the added and modified requirements is exercised or recorded as unexercised with a reason, counting scenarios from the delta files rather than from the proposal's description of them

## Workflow follow-up

- Run `/opsx:verify` once the diagnostic has been run from a session started at the greeter and from a console with no session.
- Archive, then the application launcher and the command menu, which `docs/user-experience.md` calls the element worth the most to the experience.
