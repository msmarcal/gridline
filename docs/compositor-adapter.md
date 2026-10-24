# The compositor adapter contract

Gridline treats the compositor as a replaceable component. Hyprland is the first one, and the project must not become unable to change or add compositors because of how the first one was wired in.

This document defines the contract an adapter satisfies. It deliberately does **not** describe how any particular compositor implements it: a contract written by generalizing from one compositor's interface is not a contract, it is that compositor's interface with extra steps. The Hyprland adapter is specified by `add-hyprland-session-baseline`.

The binding requirements are in `openspec/specs/compositor-abstraction/spec.md`.

## The boundary is a single path

```
packages/gridline/src/adapters/
```

**This is the compositor adapter layer. Everything outside it is compositor-neutral.**

One path, documented in one place, is what makes the isolation requirement checkable by a program rather than by a reviewer's judgment about which files look compositor-specific. Each compositor gets one subdirectory, for example `packages/gridline/src/adapters/<compositor>/`.

Outside that path, no Gridline component may contain compositor-specific command invocations, socket protocols, configuration formats, or process assumptions.

## What an adapter must report

An adapter's first obligation is to say what it can do. The session launcher asks before declaring the session up, so that a missing capability becomes a named, reported condition instead of a feature that silently does nothing.

### Required capabilities

A session does not start unless the adapter reports all of these.

| Capability | What the session needs from it |
|---|---|
| `compositor-launch` | Start the compositor and report that it is ready to accept clients, or fail with a reason |
| `session-identity` | Supply the compositor's identity token, which the launcher appends to the Gridline desktop identity |
| `output-management` | Enumerate outputs and apply a configuration, so a display change does not leave the session without usable output |
| `input-configuration` | Apply keyboard, pointer and touchpad settings from Gridline's own configuration |
| `window-control` | The tiling operations Gridline's keybindings are defined against: focus, move, resize, workspace switching |
| `idle-and-lock` | Report idle state and signal the lock component, so locking and the idle timeout work |
| `shutdown` | Terminate the compositor cleanly and release the seat on logout |

`idle-and-lock` is required rather than optional because the lock component is a fail-safe component. See [architecture.md](architecture.md).

### Optional capabilities

Absence degrades the session but does not block startup. Each must be reported as unavailable rather than left undefined.

| Capability | Consequence when absent |
|---|---|
| `screencast` | Compositor-dependent portal interfaces are unavailable; the GTK backend continues serving what it implements |
| `output-scaling` | Fractional or per-output scaling is unavailable; outputs run at integer scale |
| `touch-and-gestures` | Touch and gesture input is unavailable |

### Reporting rules

- An adapter reports a capability as available only if it can deliver it through a documented compositor interface.
- An adapter never reports a capability it implements by guessing, by parsing internal state, or by depending on behavior the compositor does not document.
- The launcher treats an unreported capability as absent. Silence is not consent.

The wire format of the report is an implementation concern owned by `add-hyprland-session-baseline`. What is fixed here is the capability set and the reporting rules.

## Only documented interfaces

**An adapter interacts with its compositor only through interfaces the compositor documents as supported.**

An adapter may not parse internal state files, rely on undocumented socket commands, or depend on the compositor's internal source layout.

When a required capability cannot be delivered through a documented interface, the correct outcome is a reported gap and an upstream issue, not a workaround. Reduced functionality is preferable to a dependency on behavior that can change without notice, because such a dependency breaks on an upstream release the project does not control and cannot predict.

A proposed implementation that needs an undocumented behavior to work does not meet this contract. That is a review failure, not a judgment call.

## Who runs the isolation check

**`add-hyprland-session-baseline` owns the mechanical adapter isolation check.** It is assigned there because that change creates both the adapter layer and the first adapter, so the check and the thing it constrains arrive together rather than one of them arriving later to find the other already violating it.

**`add-ci-vm-smoke-tests` re-runs the same check in CI**, so a later change cannot reintroduce a leak that passed review.

This matches the attribution in `openspec/changes/establish-gridline-vision/proposal.md`, which assigns the check to `add-hyprland-session-baseline` and the CI re-run to `add-ci-vm-smoke-tests`.

### How the check uses the documented path

The check uses `packages/gridline/src/adapters/` as its only input for deciding what is adapter content:

1. Partition the source tree into adapter content (inside the path) and non-adapter content (everything else).
2. Scan non-adapter content for compositor-specific references: compositor binary or command names, compositor socket or IPC protocol names, compositor configuration file formats, and compositor process names.
3. Fail on any hit, naming the file and the reference.

The check never decides whether a file "is really" compositor-specific. The path decides, which is the entire reason the boundary is a single documented path. A compositor-specific component that needs to exist belongs inside the path; a component that cannot move inside the path is a design problem the check is correctly surfacing.

## Adding a second compositor

The contract is designed so that extraction is mechanical. When a second compositor is supported:

**What changes**

- A new `packages/gridline/src/adapters/<compositor>/` directory implementing the contract above.
- The launcher's adapter selection, which resolves which adapter is active. This is a selection, not a branch: the launcher gains no knowledge of either compositor.
- The portal interface-to-backend mapping in `gridline-config`, if the new compositor needs a different backend for compositor-dependent interfaces.
- Documentation: this page's list of adapters, and the component ownership table.

**What does not change**

- Everything in `gridline-config` apart from the portal mapping. Defaults, keybindings and themes are expressed in Gridline's terms, not the compositor's.
- Everything in `gridline-tools`. The diagnostic and theme commands are compositor-neutral, and `gridline doctor` reports capability gaps by name without knowing which adapter produced them.
- The session entry, the launcher's own logic, the configuration precedence rules, and the recovery path.
- Every requirement in `openspec/specs/`, which is written in Gridline's terms throughout.

If supporting a second compositor turns out to require work outside the adapter directory beyond adapter selection, that is a violation of `specs/compositor-abstraction/spec.md` and a signal that compositor knowledge leaked across the boundary.

## The risk this does not retire

With one adapter, the boundary may encode assumptions from that one compositor that nobody notices. Defining the capability set in Gridline's own terms, rather than by generalizing Hyprland's interface, is the mitigation. It is not a proof. The risk retires when a second adapter exists, and not before.
