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

**The verification machine has no battery and no lid.** Measured on the VM: zero real `upower` devices, no `BAT*` under `/sys/class/power_supply/`, no `/proc/acpi/button/lid`, and `logind` reporting `LidClosed=false` as a default rather than an observation. Suspend is available: `/sys/power/state` offers `freeze mem` and `sleep.target` exists.

## Goals / Non-Goals

**Goals:**

- Select the locker, the idle daemon and the polkit agent, with the comparison that decided each on record.
- Make the blocking classification enforceable, so a component the project calls blocking actually blocks.
- Verify what the machine can verify, and say plainly what it cannot.

**Non-Goals:**

- Lock policy. When the screen locks and after how long is `add-gridline-configuration-defaults` and the policy layer `session-manageability` specifies.
- Reporting a missing component after the fact. `add-session-health-check` owns the diagnostic surface; refusing to start is a different mechanism.
- Implementing power management. `systemd-logind` and `upower` provide it; Gridline's obligation is to not break it and to show that it did not.

## Decisions

### `swaylock` and `swayidle` over the candidates the matrix named

`docs/component-ownership.md` names `hyprlock` and `hypridle` as candidates, and `docs/security-and-manageability.md` analyses `hyprlock`'s PAM stack at some length. The measurement above says to choose differently.

`hyprlock` links three Hyprland libraries and `hypridle` links two, among them `libhyprlang`, which is the compositor's own configuration language. `swaylock` and `swayidle` link none. All of them speak the same session lock protocol and none of them depends on its namesake compositor, so the name is branding rather than coupling in every case; what differs is the libraries.

That difference is not cosmetic for this project. `docs/security-and-manageability.md` already documents the consequence of `libhyprlang`'s `findConfig`: it returns the first configuration file it finds and stops, with no merge and no lock, which is why the manageability capability had to exist at all and why the CIS control on unoverridable screen locks has no counterpart. Selecting components that carry that library would take the configuration model the project already identified as the obstacle and put it in the fail-safe path.

The cost is that two live documents name candidates this change does not select, and both are corrected rather than left to contradict the selection.

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

## Risks / Trade-offs

- **The selection contradicts two live documents.** → Both are corrected in this change rather than left to disagree with the packages. A document naming a candidate the project did not choose is worse than one naming nothing.
- **`swaylock`'s PAM stack has not been analysed the way `hyprlock`'s was.** → It ships `/etc/pam.d/swaylock`, and reading it is part of the work rather than an assumption. If it is as broad as `hyprlock`'s `auth include login`, that is a security-review item recorded the same way.
- **Two power scenarios cannot be exercised on the verification machine.** → Battery observability and lid handling need hardware the VM does not have. They are recorded as unexercised with what would exercise them, rather than marked verified on a machine that cannot show it.
- **Blocking on components makes a missing package fatal rather than degrading.** → That is the classification the matrix already carries, and the reason is recorded there: a desktop that cannot lock must not run, and privileged actions that fail with no prompt are worse than a session that refuses.

## Migration Plan

Installing this version adds three `universe` dependencies and makes the session refuse to start if any is removed afterwards. That is a behaviour change for an installed system: a session that started yesterday with no locker will not start today. The change is what the blocking classification always meant, and the failure names the component rather than leaving the user to guess.

Rollback is the previous package version, whose launcher gates only on adapter capabilities.
