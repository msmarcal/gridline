# Architecture

How Gridline is structured, where the boundaries are, and which change delivers what.

The binding requirements are in `openspec/specs/`. This document explains the shape; the specs decide the behavior.

## Layers

Gridline is four layers inside a packaging envelope. The envelope matters as much as the layers, because the delivery rule is a structural constraint and not a distribution detail.

### Session layer

Shipped by `gridline-session`.

Owns the GDM session entry, the session launcher, and the integration glue that makes a compositor into a desktop: assembling the session environment, starting the required components, checking the adapter's capability report, and tearing the session down on logout.

This is also the layer that integrates with the rest of the Ubuntu desktop stack. It does not reimplement NetworkManager, PipeWire, BlueZ or the portals; it ensures a Gridline session reaches them the way a GNOME session does.

**Boundary:** the session layer is the only layer that knows the session is running. It never writes configuration, and it never implements a user-facing command.

### Configuration layer

Shipped by `gridline-config`.

Owns the shipped defaults beneath `/etc/xdg`, the default keybindings, the default themes, and the portal interface-to-backend mapping.

**Boundary:** configuration is data, not behavior. This layer ships no executables, and nothing in it is compositor-specific: keybindings name Gridline operations, which the active adapter carries out. Its defaults are scoped so they never affect a GNOME session. See [configuration.md](configuration.md).

### Tools layer

Shipped by `gridline-tools`.

Owns the `gridline` command, `gridline doctor`, and the theme and diagnostic commands.

**Boundary:** the tools layer runs with or without a Gridline session. A diagnostic that only works inside a healthy session is useless when it is needed, so this layer reads system state rather than depending on session state. It is entirely compositor-neutral.

### Compositor adapter layer

Shipped by `gridline-session`, confined to `packages/gridline/src/adapters/`.

Owns all compositor-specific logic, and reports the capability set the session layer checks.

**Boundary:** a single documented path, which is what makes the isolation mechanically checkable instead of a review judgment. See [compositor-adapter.md](compositor-adapter.md).

### Packaging envelope

Not a runtime layer. It is the rule that every layer above reaches users only as a Debian package, with fixed ownership boundaries and no file claimed twice.

It is listed as a structural element because `package-delivery` is a capability with real requirements, and those requirements constrain all four layers without belonging to any of them. A reader looking for where the delivery rule lives would otherwise find it nowhere.

## Capability to layer

Every capability in `openspec/changes/establish-gridline-vision/proposal.md` maps to exactly one structural element:

| Capability | Element | Why there |
|---|---|---|
| `desktop-session` | Session layer | Registration, launch, teardown and the session's user-visible existence are what this layer is |
| `platform-integration` | Session layer | The integration surface is reached by the session as it starts; no other layer talks to NetworkManager, PipeWire, BlueZ, the portals or logind |
| `compositor-abstraction` | Compositor adapter layer | The capability is the boundary, and the boundary is this layer's definition |
| `package-delivery` | Packaging envelope | The delivery rule and package ownership constrain every layer and are owned by none of them |

Two capabilities share the session layer, which is expected: a capability has one home, but a layer may be home to more than one.

The configuration and tools layers own no capability outright. They serve requirements that belong to others: configuration serves the session-scoped resolution requirements in `desktop-session` and the compositor-neutral configuration surface in `compositor-abstraction`; tools serve the diagnosability requirements spread across `platform-integration` and `compositor-abstraction`. This is why the ownership table in `proposal.md` attributes two requirements to a split rather than to a single change.

## Required session components

Most components degrade the session when absent. Two do not: their absence **blocks session startup**, because a session that runs without them fails in a way that has a security consequence rather than a missing feature.

| Component | Blocks startup | Why |
|---|---|---|
| Screen locker | **Yes** | A desktop that cannot lock must not run. The failure mode is an unlocked unattended machine, which is worse than no session at all. `specs/platform-integration/spec.md` requires that a lock component failure not leave the session exposed, and the only way to guarantee that at startup is to refuse to start |
| polkit authentication agent | **Yes** | Without an agent, privileged actions fail with no prompt and no explanation. `specs/platform-integration/spec.md` requires a missing agent to be reported rather than left to fail silently |
| Compositor, via the adapter | **Yes** | There is no session without it. Reported with the compositor named |
| Required adapter capabilities | **Yes** | The capability set in [compositor-adapter.md](compositor-adapter.md). A missing required capability is reported by name |
| NetworkManager, PipeWire, WirePlumber, BlueZ, UPower, XWayland, portals, snapd | No | Each absence is reported as a named capability gap. The session remains usable, with the corresponding feature unavailable. See [component-ownership.md](component-ownership.md) |

The rule behind the split: **block startup when running anyway would create a security exposure; degrade and report when running anyway merely loses a feature.** A blocked startup returns the user to GDM with the cause in the journal, where GNOME is still selectable, so the cost of refusing to start is one session selection rather than an unusable machine. See [recovery.md](recovery.md).

## Downstream changes

This architecture is delivered by the changes below. The authoritative attribution is the downstream table in `openspec/changes/establish-gridline-vision/proposal.md`; the change names here are the same set, with no change named in one table and missing from the other.

| Change | Delivers | Element |
|---|---|---|
| `define-debian-package-architecture` | Source package layout, binary package boundaries, file ownership rules, conffile strategy, clean removal, and the guarantee that GNOME survives install and removal unchanged | Packaging envelope |
| `define-supported-ubuntu-integration` | Component ownership matrix and package availability on the baseline, covering NetworkManager, PipeWire and WirePlumber, and BlueZ; AppArmor posture; security update path | Session layer |
| `add-gdm-wayland-session` | Session registration at GDM, launcher and teardown, desktop identity, opt-in behavior, and the recoverability guarantee | Session layer |
| `add-hyprland-session-baseline` | The adapter contract, the Hyprland adapter, capability discovery, and the mechanical adapter isolation check | Compositor adapter layer |
| `add-desktop-portal-integration` | Portal backend mapping, deterministic backend selection, XWayland support, Snap compatibility | Session and configuration layers |
| `add-session-lifecycle-integration` | Power management and suspend, lid handling, battery observability, session locking across suspend with fail-safe behavior, and the polkit authentication agent | Session layer |
| `add-gridline-configuration-defaults` | Shipped defaults, keybindings and themes; session-scoped configuration precedence; the compositor-neutral configuration surface | Configuration layer |
| `add-session-health-check` | The diagnostic command surface, capability gap reporting, and the compositor-neutral command surface | Tools layer |
| `add-ci-vm-smoke-tests` | Automated verification of session startup, integration behavior, and the adapter isolation check in CI | All elements |

Two of these were added after the original sequence was written, because the specifications grew and the sequence did not: `add-session-lifecycle-integration`, for power management, locking and polkit, and `add-gridline-configuration-defaults`, for configuration content, precedence and the compositor-neutral configuration surface. Neither had an owner before.

## Deferred requirements

Two requirements have no downstream change assigned. Both are deliberate, and both are recorded in the deferred requirements table in `proposal.md` with the same reasons given here.

| Requirement | Capability | Reason it has no owner |
|---|---|---|
| Images consume packages and add no core functionality | `package-delivery` | ISO and Remix work is out of scope for the current sequence. The requirement exists now so that image work, whenever it begins, cannot introduce image-exclusive functionality. It is a constraint on future work rather than a deliverable, so there is nothing for a scheduled change to build |
| `gridline-desktop` is reserved as a metapackage boundary | `package-delivery` | The reservation is satisfied by the change that reserves it. The metapackage itself ships once the component set it would depend on is stable, which no scheduled change reaches, so assigning it to one now would be fiction |

A requirement that appears in neither the downstream table nor this one is an attribution failure, not a deferral. Deferring is an explicit act with a stated reason.

## Related documents

- [vision.md](vision.md): what Gridline is, the baseline, and the non-goals
- [package-matrix.md](package-matrix.md): the binary packages and what each owns
- [component-ownership.md](component-ownership.md): every component, its owner, and its fallback
- [compositor-adapter.md](compositor-adapter.md): the adapter contract and the isolation boundary
- [configuration.md](configuration.md): configuration precedence and session scoping
- [recovery.md](recovery.md): what a user does when a session fails
