# Design

## Context

See proposal.md for motivation. Four measurements shape the approach.

**Every candidate is in `universe`.** There is no `main` option for a screen locker or a polkit authentication agent on this baseline, so the support gap applies whichever is chosen and the choice is made on other grounds.

**The candidates differ in what they drag in.** Measured by inspecting the archive packages without installing them:

| Package | Hyprland libraries | Session lock protocol | Ships PAM | Pulls a desktop environment |
|---|---|---|---|---|
| `hyprlock` | `libhyprgraphics4`, `libhyprlang2`, `libhyprutils10` | `ext_session_lock_manager_v1` | yes | no |
| `swaylock` | none | `ext_session_lock_manager_v1` | yes | no |
| `hypridle` | `libhyprlang2`, `libhyprutils10` | n/a | n/a | no |
| `swayidle` | none | n/a | n/a | no |
| `hyprpolkitagent` | none | n/a | n/a | no |
| `lxqt-policykit` | none | n/a | n/a | **yes**, `lxqt-session` |
| `mate-polkit` | none | n/a | n/a | **yes**, MATE components |

**Nothing enforces the blocking classification.** The launcher gates on what the adapter reports, and the adapter reports capabilities rather than components. `docs/component-ownership.md` records that the archive's user units carry `PartOf=graphical-session.target` and `Restart=on-failure`, so a failing one restarts quietly and the session comes up regardless.

**The verification machine has no battery and no lid.** Measured on the VM: zero real `upower` devices, no `BAT*` under `/sys/class/power_supply/`, no `/proc/acpi/button/lid`, and `logind` reporting `LidClosed=false` as a default rather than an observation. Suspend is **not** available either, which a first reading of the same machine got wrong. `/sys/power/state` offers `freeze mem` and `sleep.target` exists, and both were read as availability. They are not: that file lists what the kernel was built to support, not what the hardware underneath accepts. Writing to it returns an I/O error and `systemd-sleep` reports `Failed to put system to sleep. System resumed again: Operation not supported`.

## Goals / Non-Goals

**Goals:**

- Select the locker and the polkit agent, with the comparison that decided each on record.
- Give the selected component that is a systemd user unit a way to start, since declaring it installed is not starting it.
- Make the blocking classification enforceable, so a component the project calls blocking actually blocks.
- Verify what the machine can verify, and say plainly what it cannot.

**Non-Goals:**

- Lock policy, and with it the idle daemon. When the screen locks and after how long is `add-gridline-configuration-defaults` and the policy layer `session-manageability` specifies, and the component that would enforce a timeout is chosen there, alongside the policy it serves.
- Reporting a missing component after the fact. `add-session-health-check` owns the diagnostic surface; refusing to start is a different mechanism.
- Implementing power management. `systemd-logind` and `upower` provide it; Gridline's obligation is to not break it and to show that it did not.

## Decisions

### `swaylock` over the candidate the matrix named

`docs/component-ownership.md` names `hyprlock` as a candidate, and `docs/security-and-manageability.md` analyses its PAM stack at some length. The measurement above says to choose differently.

`hyprlock` links three Hyprland libraries, among them `libhyprlang`, which is the compositor's own configuration language. `swaylock` links none. Both speak the same session lock protocol and neither depends on its namesake compositor, so the name is branding rather than coupling in both cases; what differs is the libraries.

That difference is not cosmetic for this project. `docs/security-and-manageability.md` already documents the consequence of `libhyprlang`'s `findConfig`: it returns the first configuration file it finds and stops, with no merge and no lock, which is why the manageability capability had to exist at all and why the CIS control on unoverridable screen locks has no counterpart. Selecting components that carry that library would take the configuration model the project already identified as the obstacle and put it in the fail-safe path.

The cost is that two live documents name candidates this change does not select, and both are corrected rather than left to contradict the selection.

### No idle daemon is selected, and that was a correction

The proposal and an earlier draft of this design selected one, and the implementation declared `swayidle` a dependency of `gridline-session` and added it to the launcher's blocking-component check. Counting the blocking components showed the project records six and that no idle daemon is among them. Nothing in any requirement names one: `session-manageability` treats idle timeout and lock behaviour as one **policy**, and `compositor-adapter.md` requires the `idle-reporting` **capability**, which is a statement that the compositor advertises `ext_idle_notifier_v1` rather than that a daemon is listening for it.

So the selection was scope this change added to itself. It is removed: the dependency, the launcher entry, and the live-document passages that presented `swayidle` as chosen. The measurements stay on record in `docs/component-ownership.md` as candidate evidence, so `add-gridline-configuration-defaults` starts from them instead of repeating the work, and that change owns the choice because it owns the policy the daemon would enforce.

The reasoning that argued for one is still sound and is not what was wrong: manual-only locking is not locking in any useful sense. But it argues for automatic locking, which is policy plus a daemon, and policy is not this change.

### `hyprpolkitagent` despite its name

It is the only candidate that does not pull a desktop environment. `lxqt-policykit` requires `lxqt-session` and `mate-polkit` requires MATE components, and installing either would put a second desktop's session machinery on a machine whose fallback desktop is GNOME. `hyprpolkitagent` depends on Qt6 and `polkit-qt6` and carries no Hyprland library at all, so the same test that rejected `hyprlock` accepts this one.

The name invites the objection the measurement answers: what makes a component compositor-specific is what it links and what it requires, not what it is called.

### The launcher gates on components, not only on capabilities

`compositor-abstraction` already makes the session refuse to start when a required capability is unreported, and the adapter answers for capabilities. A locker is not a capability; it is a component beside the compositor, and the adapter has no business knowing it exists.

So the launcher checks blocking components itself, in the phase that already runs before anything starts. That phase was built for exactly this shape of failure: an absent file observed without running anything, reported by name, costing nothing.

The alternative was having the adapter report the locker's presence as a capability. It was rejected because it would put knowledge of a compositor-neutral component inside the compositor-specific layer, which is the boundary violation in reverse.

### A capability is not a component, and the check says so

The adapter reports `lock-signaling` when the compositor advertises `ext_session_lock_manager_v1`. That is a true statement about the compositor and says nothing about whether a locker is installed. The session today reports that capability and cannot lock.

Keeping the two checks separate keeps the failure messages honest: an absent locker is named as an absent locker, not as a missing capability, and a compositor that does not implement the protocol is named as that.

### A Gridline-owned user target, because declaring a unit is not starting it

Declaring `hyprpolkitagent` a dependency puts the file on disk. Three separate things then stop it from running, and all three were measured on the baseline VM rather than assumed:

- **The unit is shipped disabled.** `dpkg -L hyprpolkitagent` lists `/usr/lib/systemd/user/hyprpolkitagent.service` and no symlink, so the unit's `[Install] WantedBy=graphical-session.target` is inert without an `enable` that nothing runs. Confirming that by hand means listing **both** unit roots, `/usr/lib/systemd/user/graphical-session.target.wants/` and `/etc/systemd/user/graphical-session.target.wants/`, because an `enable` writes into the second. A first pass read only the first and missed a stale experimental enable sitting in the second, which is what invalidated the first run of task 7.6.
- **`graphical-session.target` cannot be started.** It carries `RefuseManualStart=yes`, so the launcher cannot start it and neither can anything else except a dependency.
- **The unit is conditioned on `WAYLAND_DISPLAY`**, which the systemd user manager does not have: it inherits nothing from the display manager. A failed `ConditionEnvironment` is a **skip**, not a failure, so the component would quietly not run and nothing would be marked wrong.

**Only the agent needs this.** The other unit among the blocking components, the compositor's portal backend, is `Type=dbus` with a D-Bus activation file and **no `[Install]` section at all**, so it starts when the portal frontend asks for its bus name. Measured in a running session it is `active` with `WantedBy=` and `TriggeredBy=` both empty. The target therefore wants one unit rather than two, and a verification pass that assumed the two units behaved alike was wrong about the portal backend.

`gridline-session` therefore ships `/usr/lib/systemd/user/gridline-session.target`. It `Wants=` the unit the session needs running, and it is `BindsTo=graphical-session.target` and `Before=` it, so starting the session target pulls the standard target in as a dependency, which `RefuseManualStart` permits. The launcher sets `WAYLAND_DISPLAY` in the user manager, starts the session target, and then checks that each unit the target wants is running.

Three alternatives were weighed:

- **Ship the enable symlink** under `graphical-session.target.wants`. Rejected: that target belongs to every graphical session, so the agent would start under GNOME too. Installing Gridline must leave the fallback desktop exactly as it was, which is what makes removal safe.
- **Have the launcher start `/usr/libexec/hyprpolkitagent` directly.** Rejected: it reimplements supervision the archive's unit already provides, and it means that when the next component arrives there is still no mechanism for it.
- **Repeat the unit names in the launcher** so it knows what to check. Rejected: the target already declares them, and a blocking classification with two sources of truth is the shape of defect this project keeps finding. The launcher reads the list back with `systemctl --user show -p Wants --value`, which reports exactly the `Wants=` entries and keeps `BindsTo=` separate, confirmed on the VM.

The launcher reads unit state to do that check, which the pre-flight component check deliberately does not. The two questions are different and so is the moment: before anything runs, unit state is no evidence a component is installed, because `Restart=on-failure` makes a broken one look alive; after the target has been started, unit state is the only thing that says whether the component is running.

**The target carries no `Before=graphical-session.target`, and an earlier version did.** That version did not work, and the way it failed is worth recording because it was invisible to every check except the one this change added. A target with `DefaultDependencies=yes` gains an implicit `After=` on everything in its `Wants=`; the agent's archive unit is `After=graphical-session.target`; so `Before=graphical-session.target` on the session target closed a cycle. systemd breaks a cycle by deleting a job, and it deleted the agent's: `gridline-session.target: Job hyprpolkitagent.service/start deleted to break ordering cycle`. The target was then reached, the session looked ready, and the component had never run. The launcher's unit check refused the session and named the unit, which is the only reason this was found at the greeter rather than months later as a missing password prompt.

Removing the `Before=` leaves the order the dependencies already imply, and it is the order wanted: `graphical-session.target`, then the agent, then the session target through the implicit `After=`. A start of the session target therefore does not return until the units it wants have been tried, so the check that follows is not racing the jobs it is checking.

**The unit check has a limit, and establishing it took two attempts.** It catches a unit that never started. It does not catch one that starts and then keeps crashing: these units are `Type=simple` and so are active from the fork, and the check runs immediately after the start. The window was measured, not estimated. The target was reached at `16:53:20.710`; the agent's first core dump landed at `16:53:24.802`, `status=6/ABRT`; `Restart=on-failure` then cycled it four times in six seconds. Four seconds in which the check sees a healthy unit.

A first write-up of this said the opposite, that the agent exits 0 and the restart never fires, on the strength of `ExecMainStatus=0` and `NRestarts=0` read from `systemctl --user show`. Those values were read from a **different user manager instance** than the one that ran the unit: with no graphical session open, `user@1000.service` stops shortly after the shell that started it exits, so the second command spoke to a fresh manager reporting the defaults of a unit it had never run. The journal of the manager that did run it says `code=dumped`. The lesson is the one this project keeps relearning: the defect was in what the instrument was measuring, not in the thing measured.

Extending the launcher to cover the window was rejected: a startup gate cannot promise a daemon stays alive, any delay it waited would be arbitrary, and `add-session-health-check` owns observing a session over time.

**The target names a component outside the adapter layer**, and the isolation check does not flag it, which is the recorded rule rather than an exception: a component is compositor-specific because of what it links and requires, and `hyprpolkitagent` links no Hyprland library and pulls no desktop environment. The check did fire once here, on an earlier version of the target's own comment that restated the measurement and so wrote the compositor's name. The comment now points at the document instead, which is the better shape anyway.

## Risks / Trade-offs

- **The selection contradicts two live documents.** → Both are corrected in this change rather than left to disagree with the packages. A document naming a candidate the project did not choose is worse than one naming nothing.
- **`swaylock`'s PAM stack has not been analysed the way `hyprlock`'s was.** → It ships `/etc/pam.d/swaylock`, and reading it is part of the work rather than an assumption. If it is as broad as `hyprlock`'s `auth include login`, that is a security-review item recorded the same way.
- **Two power scenarios cannot be exercised on the verification machine.** → Battery observability and lid handling need hardware the VM does not have. They are recorded as unexercised with what would exercise them, rather than marked verified on a machine that cannot show it.
- **Blocking on components makes a missing package fatal rather than degrading.** → That is the classification the matrix already carries, and the reason is recorded there: a desktop that cannot lock must not run, and privileged actions that fail with no prompt are worse than a session that refuses.
- **The launcher now refuses the session when a unit fails to start, not only when a file is missing.** → That is the same classification applied one step later, and without it the blocking rule is unenforced again for the component that is a unit rather than a program. The failure names the unit, and `Wants=` rather than `Requires=` keeps a failing unit from taking `graphical-session.target` down before the launcher can say which one it was.
- **Gridline now owns a systemd unit, which the proposal said it would not.** → The proposal was written before the agent's unit was read, and it turned out that declaring the dependency started nothing at all. Shipping a target is the smallest mechanism that fixes it without touching an archive unit or enabling one, and the rule the proposal was protecting, that Gridline modifies no system configuration outside paths it owns, is intact: the target is a new file under a Gridline-specific name.

## Migration Plan

Installing this version adds two `universe` dependencies and makes the session refuse to start if either is removed afterwards. That is a behaviour change for an installed system: a session that started yesterday with no locker will not start today. The change is what the blocking classification always meant, and the failure names the component rather than leaving the user to guess.

It also adds a file under `/usr/lib/systemd/user/`, which the user manager picks up without any maintainer script: a target that is only started by the launcher has nothing to enable, so the package installs no symlink and runs no `systemctl`.

Rollback is the previous package version, whose launcher gates only on adapter capabilities and which starts no target. Nothing in the archive was modified, so rolling back leaves no trace.
