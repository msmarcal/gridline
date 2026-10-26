# Design

## Context

See proposal.md for motivation. Four things are already settled and are inputs here rather than decisions.

**The capability set is fixed.** `docs/compositor-adapter.md` names nine required capabilities and two optional ones, with the reason each required one is required. This change does not reopen that list; it defines how an adapter says which of them it has.

**The boundary is fixed.** `packages/gridline/src/adapters/` is the single documented path, and the isolation check decides by that path alone.

**The launcher's shape is fixed.** It checks required components in order, reports the first missing one through `logger -t gridline-session`, and exits 1 so the display manager regains the seat. Its third branch is the one this change replaces.

**The baseline can satisfy the contract.** Measured against `hyprland` 0.53.3+ds-4 on Ubuntu 26.04.1 amd64:

| Capability | Documented interface |
|---|---|
| `compositor-launch`, `session-identity`, `input-configuration`, `window-control`, `shutdown` | `hyprctl` |
| `lock-signaling` | `ext_session_lock_v1` |
| `idle-reporting` | `ext_idle_notifier_v1` |
| `output-management` | `zwlr_output_manager_v1` |
| `screencast` | `zwlr_screencopy_manager_v1` and `hyprland_toplevel_export_manager_v1`, **and** the compositor's portal backend declaring `org.freedesktop.impl.portal.ScreenCast` |
| `output-scaling` (optional) | `wp_fractional_scale_manager_v1` |
| `touch-and-gestures` (optional) | `zwp_pointer_gestures_v1` |

## Goals / Non-Goals

**Goals:**

- A wire format simple enough that a shell adapter is natural and a reimplementation in another language changes nothing for the session.
- A session that starts, and a failure path that still names its cause.
- An isolation check that runs before the package is built, like the ownership gate it joins.

**Non-Goals:**

- Shipping Hyprland configuration. The session starts with Hyprland's own defaults; keybindings and themes belong to `add-gridline-configuration-defaults`.
- Driving the lock component. `lock-signaling` reports that the compositor implements the lock protocol, not that a locker is installed or chosen.
- A second adapter, or CI.

## Decisions

### The query is an argument, and the answer is lines on stdout

The launcher invokes `<adapter> capabilities`. The adapter writes one capability name per line to stdout and exits 0, or exits non-zero to mean its answer is not usable.

Alternatives considered. **JSON** was rejected because the launcher is `/bin/sh` and parsing JSON there means adding a dependency or writing a parser, both worse than the problem. **A key=value environment dump** was rejected because it invites an adapter to report a capability as present-but-degraded, which the contract does not have: a capability is reported or it is not. **A file the adapter writes** was rejected because it introduces a staleness question the process-exit answer does not have.

One capability per line is parseable with `while read`, greppable, diffable in a bug report, and legible when a user runs the adapter by hand. The exit status carries the only other thing the session needs: whether the answer can be trusted at all.

### Silence and failure are the same answer

A capability absent from the output and an adapter that exits non-zero both mean the capability is unavailable. The contract already says silence is not consent; this extends it so that a partial answer cannot be mistaken for a complete one. An adapter that crashes halfway through printing has not reported the capabilities it managed to print.

The cost is that a broken adapter and an honest minimal adapter are indistinguishable to the session. That is acceptable because both outcomes are the same: the session does not start and the first missing required capability is named. The adapter's own stderr reaches the journal for anyone diagnosing which of the two it was.

### The capability check happens in two phases, because one phase cannot work

The first draft decided the adapter reports a capability when the compositor advertises the protocol that serves it. Implementation showed that cannot be done at the moment the contract asks for it. The capability query runs before the compositor starts, and on the baseline a Hyprland that is not running answers nothing: `hyprctl` exits non-zero without an instance signature, and `Hyprland --help` aborts before parsing arguments because `XDG_RUNTIME_DIR` is unset. There is no documented interface that reports protocol support from an idle installation.

So the launcher asks twice. `capabilities` before launch reports what the adapter can establish from installed files alone, and its exit status is the early gate: an adapter that already knows it cannot proceed says so before anything starts. `start` launches the compositor. `confirm` then asks the same question against the running compositor, and that answer is the gate for required capabilities.

**Pre-launch and post-launch is not the same split as optional and required.** A capability can be required and only confirmable after launch, and `lock-signaling` is exactly that: it is blocking, because a desktop that cannot be shown to lock must not run, and it is unknowable until the compositor is up to advertise `ext_session_lock_v1`. Reading "confirmed later" as "matters less" would contradict the blocking classification `define-supported-ubuntu-integration` recorded. The phase says when the answer is available, not how much the answer counts.

Three alternatives were rejected. **Launching first and asking afterwards without a pre-launch phase** puts the only check at a point where it can no longer prevent what it exists to prevent, which is the defect the ownership gate's anchor history already taught this project. **Reading protocol support out of the compositor binary** is depending on internal layout, which the contract forbids; it was legitimate as a planning measurement and is not legitimate as a mechanism. **Narrowing the capability set to what is checkable early** reopens a list `docs/compositor-adapter.md` owns and this change said it would not reopen.

### The adapter reports a capability only if its interface is present, checked at the phase that can establish it

For capabilities served by a Wayland protocol, the Hyprland adapter reports the capability under `confirm`, when the running compositor advertises that protocol. For capabilities served by `hyprctl`, it reports them under `confirm` too, when `hyprctl` answers against the running instance. Under `capabilities` it reports only what installed files establish, which on this baseline is the portal half of `screencast` and the presence of the compositor binary.

The alternative was a static list: Hyprland 0.53 supports these, so report them. That was rejected because it makes the adapter lie after an upstream change, which is the failure the contract's "never report a capability it implements by guessing" rule exists to prevent. A version-pinned list is a guess with a date on it.

### `screencast` is reported only when both halves are present

`docs/compositor-adapter.md` defines `screencast` as serving the compositor-dependent portal interfaces, not as the compositor advertising a capture protocol. Those are different claims, and the first draft of this design confused them by listing only the compositor protocols as what backs the capability.

The adapter reports `screencast` when the compositor advertises the screencopy protocols **and** the compositor's portal backend declares `org.freedesktop.impl.portal.ScreenCast` in its `.portal` file with an installed D-Bus service. On the baseline that file is `/usr/share/xdg-desktop-portal/portals/hyprland.portal`, declaring `org.freedesktop.impl.portal.desktop.hyprland`. Both halves are readable at query time without running anything.

Reporting on the compositor half alone would have the adapter claim a capability it cannot deliver without a component it does not check, which is what the contract's rule against reporting a guessed capability forbids. It would also make `screencast` the one required capability whose absence is discovered after the session starts, which is the failure `docs/compositor-adapter.md` says is worse than not starting.

**The backend this resolves to is reached through the compositor's name, not Gridline's.** `hyprland.portal` lists `UseIn=wlroots;Hyprland;sway;Wayfire;river;` with no `Gridline`, and Gridline's identity is `Gridline:Hyprland`, so the frontend matches on the second name. `docs/component-ownership.md` already records that this is a working path by accident and that `gridline-portals.conf` is what makes it deliberate. That file belongs to `add-desktop-portal-integration`; this change depends on the accidental path and says so rather than leaving a later reader to discover it.

### The launcher starts the compositor through the adapter, not directly

After the capability check passes, the launcher invokes `<adapter> start`. The launcher never names Hyprland. This is what keeps the isolation check from failing on the launcher itself, and it is the mechanism behind the `compositor-abstraction` requirement that supporting a second compositor is confined to selecting which adapter is active.

### The isolation check runs as a build gate beside the ownership gate

`debian/rules` already runs `tools/check-path-ownership` before `dh_builddeb`. The isolation check joins it there. Anchoring both at the same point keeps one answer to "when are the gates evaluated", and that point is the one the ownership gate's own history established as the only place where every staging tree is complete.

The check partitions the source tree by `packages/gridline/src/adapters/` and scans everything outside it for compositor references. Its reference list starts from what this change actually introduces: the compositor binary name, `hyprctl`, the Hyprland protocol names in the table above, and the Hyprland configuration file name. A check whose list is empty passes trivially and proves nothing.

### `Depends`, not `Recommends`, for both new dependencies

`hyprland` and `xdg-desktop-portal-hyprland` both block session startup, so the policy in `docs/packaging.md` makes them dependencies. `xdg-desktop-portal-hyprland` is blocking for the reason `docs/compositor-adapter.md` records: without it the adapter cannot report `screencast`, and `screencast` is required because the GTK backend does not implement screen capture on Wayland.

Both are in `universe`. The support gap is stated where the dependency is declared, which for a package is its long description, because that is what an administrator reads before installing.

## Risks / Trade-offs

- **The adapter reports `lock-signaling` while no locker exists.** → The capability is about the compositor implementing the lock protocol, not about a locker being installed. The locker is a separate blocking component, classified in `docs/component-ownership.md` and owned by `add-session-lifecycle-integration`. A session that starts here is not yet a session that can lock, and saying so is this change's obligation.
- **Protocol detection can be wrong about what the compositor actually serves.** → Advertising a protocol is not the same as implementing it correctly. The adapter reports what it can establish through a documented interface, which is the strongest claim available without running the compositor; behavior verification belongs to the scenarios, not the capability query.
- **A session that starts with Hyprland's defaults will not look like Gridline.** → Expected, and worth stating plainly so nobody reads the first running session as the product. `add-gridline-configuration-defaults` is what makes it Gridline's desktop rather than Hyprland's.
- **The isolation check's reference list can go stale as the adapter grows.** → The check fails closed on the references it knows and is extended by the change that adds a compositor-specific mechanism. `add-ci-vm-smoke-tests` re-runs it, which catches a leak that passed review but not one the list never covered.

## Migration Plan

Installing this version adds two `universe` dependencies and makes a previously unselectable session selectable and startable. Removing the Gridline packages returns the system to stock, as before; the two archive packages remain installed because dpkg does not remove dependencies, which is ordinary behavior and not Gridline state.

Rollback is reinstalling the previous package version: the session entry's `TryExec` still resolves, the launcher reports the missing adapter contract, and the user is returned to GDM. The previous version's failure path is the rollback path.
