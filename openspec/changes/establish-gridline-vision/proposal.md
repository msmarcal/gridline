# Proposal: establish-gridline-vision

## Why

Project Gridline is an optional Wayland tiling desktop session for Ubuntu, delivered as Debian packages on top of a stock Ubuntu LTS installation. It is not a distribution fork and not a replacement for Ubuntu Desktop.

The project has a clear product idea and a set of hard constraints, but nothing written down that a later change can be checked against. Without a recorded foundation, every downstream change would have to restate the baseline, the delivery rule, and the fallback guarantee, and would drift from them the first time a constraint became inconvenient.

This change records that foundation: what Gridline is, what it must never break, how it is delivered, and where the compositor boundary sits.

## What Changes

This change ships documentation and specifications only. No package is built, no behavior is implemented, and no hardware validation is performed. Every implementation task belongs to a downstream change named below.

- Establishes the product vision, the supported baseline of Ubuntu 26.04 LTS on x86_64, and the project-wide non-goals.
- Adds four capability specifications that later changes are verified against.
- Records the structural decisions that constrain all later work: one source package producing multiple binary packages, Gridline defaults scoped to the Gridline session, deterministic portal backend selection, and a single documented compositor adapter path.
- Documents the recovery path for a failed Gridline session.

## Capabilities

### desktop-session

The session as a user-visible artifact: how it is offered at the login screen, that it is opt-in and never becomes the default, that the stock GNOME session survives installation unchanged, and that a failed Gridline session is recoverable without a working Gridline desktop.

### package-delivery

The delivery rule and its consequences: all functionality arrives through Debian packages, nothing is fetched at install time, package boundaries are fixed, no file is claimed twice, local configuration changes survive upgrade, removal returns the system to a stock desktop, and a future image consumes the same packages rather than adding functionality of its own.

### platform-integration

The integration surface a Gridline session must satisfy to count as a usable Ubuntu desktop: NetworkManager, PipeWire and WirePlumber, BlueZ, XDG Desktop Portals with deterministic backend selection, Snap applications, power management covering suspend and lid handling, session locking that survives suspend, a polkit authentication agent, XWayland, AppArmor confinement, and Ubuntu security updates.

### compositor-abstraction

The boundary that keeps the compositor replaceable: compositor-specific logic confined to a single documented adapter path, no dependency on undocumented compositor internals, capability discovery rather than assumption, and a user-facing surface expressed in Gridline's own terms.

## Impact

- **Affected specs:** four new capabilities, listed above. No existing capability is modified, because none exists yet.
- **Code:** none changed. The repository contains OpenSpec scaffolding and a `.gitignore` written for `dpkg-buildpackage`.
- **Breaking changes:** none. Nothing ships in this change.
- **Users:** none yet. Gridline is additive and removable by construction, and the stock GNOME session remains installed, selectable and supported at all times.

## Out of scope

- Choosing the contents of Gridline defaults, keybindings, or themes. The ownership boundary is specified here; the content is a later change.
- Declaring APT dependency relationships between packages. This change says which package owns what, not what each one depends on.
- Selecting a status bar, launcher, or shell.
- Omarchy visual parity, a custom shell, per-device hardware automation, an application catalogue, and AI features. These are project-wide non-goals.
- Any ISO or Remix work. A future image consumes the packages defined here.

## Downstream changes

Every requirement specified in this change is the responsibility of exactly one of the changes below, except for the deferred requirements declared at the end of this section.

| Change | Delivers |
|---|---|
| `define-debian-package-architecture` | Source package layout, binary package boundaries, file ownership rules, conffile strategy, clean-removal behavior, and the guarantee that the stock GNOME session survives Gridline install and removal unchanged |
| `define-supported-ubuntu-integration` | Component ownership matrix and package availability on the supported baseline, covering NetworkManager, PipeWire and WirePlumber, and BlueZ; AppArmor posture; security update path |
| `add-gdm-wayland-session` | Session registration at GDM, session launcher and teardown, desktop identity, opt-in behavior, and the fallback guarantee that a failed Gridline session is recoverable without Gridline |
| `add-hyprland-session-baseline` | Compositor adapter contract, the Hyprland adapter, capability discovery, and the mechanical adapter isolation check |
| `add-desktop-portal-integration` | Portal backend mapping, deterministic backend selection, XWayland support, Snap application compatibility |
| `add-session-lifecycle-integration` | Power management and suspend, lid handling, battery observability, session locking across suspend with fail-safe behavior, and the polkit authentication agent |
| `add-gridline-configuration-defaults` | Shipped defaults, keybindings and themes; session-scoped configuration precedence; and the compositor-neutral user-facing configuration surface |
| `add-session-health-check` | The diagnostic command surface, capability gap reporting, and the compositor-neutral command surface |
| `add-ci-vm-smoke-tests` | Automated verification of session startup, integration behavior, and the adapter isolation check in CI |

Two changes in this table are new. `add-session-lifecycle-integration` exists because power management, locking and polkit were added to `platform-integration` after the original sequence was written, and no existing change covered them. `add-gridline-configuration-defaults` exists because the configuration content and the configuration precedence rule had no owner, which also left the compositor-neutral configuration surface unattributed.

Two attributions are deliberately split along a boundary that is not a gap. The recoverability requirement in `desktop-session` is owned by `add-gdm-wayland-session`, which delivers the guarantee itself; `add-session-health-check` makes the failure diagnosable but does not own the guarantee, because recovery must work when no Gridline component runs. The compositor-neutral surface requirement in `compositor-abstraction` is owned by `add-gridline-configuration-defaults` for configuration and `add-session-health-check` for commands, with the adapter isolation check in `add-hyprland-session-baseline` verifying both.

The requirement that the stock GNOME session remains installed and supported is owned by `define-debian-package-architecture` rather than by a session change, because its two scenarios are install-time and removal-time behavior. What protects GNOME is the packaging: file ownership rules that prevent a Gridline package from claiming a file another package owns, and clean removal that returns the system to a stock desktop. No behavior in the Gridline session itself can provide that guarantee.

The adapter isolation check is assigned to `add-hyprland-session-baseline`, where the adapter is created, and re-run in CI by `add-ci-vm-smoke-tests`.

### Deferred requirements

Two requirements in this change have no downstream change assigned, by decision rather than by omission.

| Requirement | Capability | Reason |
|---|---|---|
| Images consume packages and add no core functionality | `package-delivery` | ISO and Remix work is out of scope for the current sequence. The requirement exists now so that image work, whenever it begins, cannot introduce image-exclusive functionality. It is a constraint on future work, not a deliverable. |
| `gridline-desktop` is reserved as a metapackage boundary | `package-delivery` | The reservation is satisfied by this change. The metapackage itself is delivered when the component set it would depend on is stable, which no scheduled change reaches. |

A deferred requirement must be listed here explicitly. Leaving a requirement out of both tables is an attribution failure, not a deferral.

