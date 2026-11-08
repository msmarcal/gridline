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
- [x] 2.3 **It does not survive, measured at the only moment it could be: after logging out and before logging back in.** `/run/user/1000` does not exist at the greeter. The user's sessions are gone, the user manager stopped with them, and the runtime directory went with it, taking the record.

  This was doubted before it was measured, and the doubt was reasonable: a runtime directory outlives any single session while the user has another, so "scoped to the session" could have meant "scoped to the user manager". It is scoped by the manager's lifetime, and with the last session gone so is the manager.

  **The condition that makes this true is `Linger=no`**, which is the default and is not a guarantee. With lingering enabled the manager outlives every session and the record would outlive them too, which would make a machine stop trying a component that failed in a session the user has long left. The project has been bitten by exactly this before, which is why the launcher unsets its own environment markers at teardown rather than relying on the manager dying
- [x] 2.4 **Recording cannot refuse a session, by construction and by measurement.** In the code, the only path that ends a launch is `report_missing`, whose every call site exits 1 and which this change did not touch; `record_component` ends in `|| true` and has no exit path of its own, so a recorder that is missing, unwritable or broken changes nothing about whether a session starts.

  Measured in a session that genuinely had a failed component: the launcher's journal identifier carries no entries at all for this login, and the desktop came up. A session that refused would have said so under that identifier, which is the one thing the launcher always does before giving up

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
- [x] 3.4 Said per component and verified against the session's own wiring, on the first session started by the launcher that records blocking components:

  ```
  bar started no          locker started yes
  notifications started no   polkit-agent started yes
  wallpaper failed no
  ```

  Cross-checked: `gridline-session.target` wants `hyprpolkitagent.service`, which is the polkit agent, and `gridline-shell.target` wants the three that read `no`.

  **One of the five is not in any target, and that is worth saying rather than glossing.** The screen locker is checked by the launcher as an installed program, not as a unit it waits for, so its severity comes from the launcher's own list of what a session requires rather than from a target's `Wants=`. The cross-check therefore covers four of five against the targets and the fifth against the launcher, and both are Gridline's own declarations rather than a list written inside the command
- [x] 3.5 Run with no session and no `XDG_RUNTIME_DIR` at all: it answers, in the same words as the empty-record case, and exits 0. It does not error, and it does not report a healthy session either, which are the two ways this could have been wrong

## 4. Stopping the hold

- [x] 4.1 **0 starts in two minutes, against 59.** The component that cannot start on the verification VM now reaches `failed` with `NRestarts=5`, the journal says `Start request repeated too quickly`, and nothing revives it. The three moved from `Upholds=` on the session target to `Wants=` on `gridline-shell.target`, which the session target wants; `systemctl --user show` reads back `Wants=hyprpolkitagent.service gridline-shell.target` and `Upholds=gridline-idle.service`, so the launcher's blocking check still sees one service and a target that is active whatever its own wants did
- [x] 4.2 Killed and returned, both of them, with Gridline's configuration on the command line rather than the bare binary. This was the thing most likely to break and it needed the drop-in to carry `Restart=always`: the notification daemon's shipped unit declares no `Restart=` at all, and both components came back before this change only because `Upholds=` revived them. A clean `SIGTERM` is not a failure, so `on-failure` would not have been enough either
- [x] 4.3 **Tried again, from zero.** The component given up on in the previous session was started 5 times in this one, reached its limit at 01:02:04 against a compositor that started at 01:01:45, and stopped. The record carries `at=2026-11-08T01:02:02Z`, which is this session rather than a stale entry: the previous record went with the runtime directory when the user manager stopped, measured at the greeter in task 2.3.

  So a machine that cannot run a component today tries it again tomorrow, and the thing that expires the decision is the session ending rather than any code written to expire it
- [x] 4.4 The component that the session stopped holding is the one the diagnostic names: after it reached `failed` with the start limit exhausted, `gridline doctor` reports `wallpaper  failed  (the desktop works without it, with less)` and exits 1.

  This is the pairing the change exists for. Giving up without reporting would have been a worse state than the loop it replaced, because a loop is at least visible in the journal, and a component silently abandoned is a desktop missing a piece with nothing anywhere saying so

## 5. Verification in a session

Execution note: this group needs the session running on the VM, started from the greeter, and it needs the wallpaper daemon left unmasked, since the machine's inability to run it is the genuine instance of the condition this change reports.

- [x] 5.1 The whole path, on the machine's own failing component rather than a staged one: logged in from the greeter, the wallpaper daemon failed 5 times and stopped, the session recorded `state=failed blocking=no`, and `gridline doctor` named it `wallpaper  failed  (the desktop works without it, with less)` and exited 1. Zero ordering cycles in the manager that started with the session
- [x] 5.2 **Unchanged, and this round had more to prove than the last one.** Three things that did not exist before could have broken it: a new target, `Restart=always` in each drop-in where the archive's units declare either `on-failure` or nothing, and an `ExecStopPost` that writes a record whenever a unit stops.

  Measured against each: the three components `inactive` with **0 restarts** and no processes, both Gridline targets inactive, **no failed unit of any kind**, and `/run/user/1000/gridline/` does not exist, so nothing of Gridline's wrote session state in a session that is not Gridline's. `Restart=always` cannot loop where nothing wants the unit, and `ExecStopPost` cannot fire where nothing stopped.

  The rest is as before: GNOME's own daemon owns the notification name, `XDG_CONFIG_DIRS` carries no Gridline directory, `GRIDLINE_CONFIG` is absent, and the person using the machine reports nothing different.

  **The baseline was taken between sessions rather than assumed**, at the greeter with the previous session ended: no runtime directory, no component process, no compositor, and `graphical-session.target.wants` holding only the VM's own guest agent. Anything found afterwards was therefore the Ubuntu session's doing rather than a leftover
- [x] 5.3 The desktop is usable with the gap present, which is what the person using the machine is doing while this is written: the bar draws, notifications work, and the session did not refuse to start over a component that cannot run here. The reporting this change adds did not quietly turn a degrading component into a blocking one, which is the way it could have gone wrong without anybody noticing until a session refused

## 6. Coherence

- [x] 6.1 **Narrowed rather than removed, because this change closes part of the promise and not the whole of it.** The task assumed removal; counting the cells said otherwise. Of the fourteen, **three** are components the session starts and now reports, the bar, the notification daemon and the wallpaper daemon, joined by the locker and the polkit agent. The other eight are platform integrations, NetworkManager, PipeWire, WirePlumber, BlueZ, UPower, XWayland, snapd and the two portal rows, which Gridline integrates with and does not start.

  Reporting those would mean a Gridline command inspecting the state of units other packages own, which is the coupling the design of this change refuses by name. So the paragraph now says which rows are true, which are owed, and why the remainder is a different problem rather than an oversight.

  The counts were taken again rather than trusted: **14 cells on three pages, 5 further mentions in prose, 3 scenarios in `openspec/specs/`**. The wrong number this change was created to correct, seventeen cells across five documents, is gone from the page
- [x] 6.2 **Enumerated, and the answer is three of eleven.** The fourteen cells are: eight platform integrations, PipeWire, WirePlumber, BlueZ, UPower, XWayland, snapd and the two portal rows; the three components the session starts; one summary row naming many at once; and two rows that name this change and `gridline-tools` rather than a component.

  | Cell | What reports it |
  |---|---|
  | Status bar, Notification daemon, Wallpaper daemon | `gridline doctor`, from the record the session writes |
  | PipeWire, WirePlumber, BlueZ, UPower, XWayland, snapd, the two portals, and the summary row | nothing |

  The screen locker and the polkit agent are reported too and are not on this list, because their rows read "Session does not start": they are blocking, and this list is the degrading ones.

  So a cell that no longer applies is a cell to change, as the task says, and eight of them still apply. `docs/component-ownership.md` says which are which and why the remainder is a different problem rather than an oversight, which is task 6.1's output and is what makes leaving them honest rather than quiet
- [x] 6.3 The document already had a step promising `gridline doctor`, written before the command existed. It now describes what the command does: the components it names, that it answers without a session running, that nothing recorded reads differently from everything healthy, and its exit codes. It also says plainly what it does not cover, pointing at the page that says why.

  **The worked example in it is output that was observed, not output that was composed.** A first draft showed the locker and the polkit agent in the list; those lines have not been seen, because the session on the verification machine started before the launcher that records them. The example is now the three lines that were measured, with a sentence for the two that appear on a session started afterwards, and that sentence is what the next login verifies
- [x] 6.4 **Fourteen scenarios, counted from the two delta files**: six added under `session-manageability`, eight in the modified `desktop-session` requirement. Eleven are exercised, two are not, and one is exercised by this change only in part.

  | Scenario | Where | State |
  |---|---|---|
  | A missing component is nameable | 3.2 | **Exercised.** `wallpaper failed`, under the Gridline name, with no unit name anywhere in the record |
  | A healthy session says so | 3.3 | **Not exercised.** This machine cannot run the wallpaper daemon, so a session with every component running has never existed here to be reported. The empty-record and failing cases both read distinctly, which is the half that could be shown |
  | The report distinguishes what refuses a session from what degrades it | 3.4 | **Exercised**, and cross-checked against the session's own targets for four of five components and against the launcher's list for the locker, which is in no target |
  | The diagnostic reads a record the session wrote | 3.2 | **Exercised.** Zero occurrences of any archive unit name in the command, and the only units it reads are Gridline's two targets |
  | The diagnostic works when the session does not | 3.5 | **Exercised** with no session and no `XDG_RUNTIME_DIR`: it answers and exits 0 |
  | Reporting a gap never refuses a session | 2.4, 5.1 | **Exercised** in code and in a session that had a failed component and started anyway |
  | The state of the machine is visible | inherited | Not this change's. Carried unchanged from the requirement it modifies |
  | A notification reaches the user | inherited | Not this change's |
  | The desktop has a background | inherited | Not this change's, and still unexercised for the reason its own change recorded |
  | Each component is kept running | 4.2 | **Exercised**, and the wording it gained matters: it now covers a component that *has run* and stops, which is what `Restart=always` in the drop-in delivers |
  | A component that cannot start is not held forever | 4.1, 4.3, 4.4 | **Exercised.** 0 starts in two minutes against 59, reaching `failed` and staying, reported by the diagnostic, and tried again from zero in the next session |
  | An absent component degrades and does not block | 5.1, 5.3 | **Exercised** on the machine's own failing component rather than a staged one |
  | Their configuration is a shipped default a user can decline | inherited | Not this change's |
  | Only the Gridline session is affected | 5.2 | **Exercised**, against three mechanisms this change introduced that could each have broken it |

  The two that are not exercised are the healthy-session report, which needs a machine that can run every component, and the background, which is the same machine limit recorded by the change that shipped it

## Workflow follow-up

- Run `/opsx:verify` once the diagnostic has been run from a session started at the greeter and from a console with no session.
- Archive, then the application launcher and the command menu, which `docs/user-experience.md` calls the element worth the most to the experience.
