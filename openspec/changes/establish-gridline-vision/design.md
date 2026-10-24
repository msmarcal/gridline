# Design

## Context

See proposal.md - Why. The repository is empty apart from OpenSpec scaffolding and a `.gitignore` already written for `dpkg-buildpackage`/debhelper with per-package rules under `packages/*/debian/`.

Constraints that shape everything below:

- **Baseline is fixed**: Ubuntu 26.04 LTS, x86_64, GDM.
- **Gridline is a guest on someone else's desktop.** The stock GNOME session must keep working byte-for-byte after Gridline is installed (see `specs/desktop-session/spec.md`). This is the sharpest constraint in the design: most of the decisions below exist to keep Gridline's configuration and environment from leaking outside its own session.
- **Delivery is packages only** (`specs/package-delivery/spec.md`), so anything that would normally live in a setup script has to become package content or session startup logic.
- **The compositor is replaceable** (`specs/compositor-abstraction/spec.md`), while the only compositor that exists today is Hyprland. The abstraction has to be real enough to honor the spec without inventing a second implementation to justify it.

## Goals / Non-Goals

**Goals:**

- A repository and package layout that satisfies the package boundaries without creating version skew between the four binary packages.
- A concrete placement for the compositor adapter boundary, so later changes know what counts as "outside the adapter."
- An environment and configuration strategy that keeps Gridline defaults invisible to the GNOME session.
- A deterministic answer for portal backend selection, which is where the Hyprland backend and the GTK backend otherwise collide.

**Non-Goals:**

- Choosing the contents of the Gridline defaults, keybindings, or themes. The boundary is specified here; the content is a later change.
- Specifying the `gridline doctor` check list. Only the requirement that capability gaps be reportable is settled here.
- Declaring APT dependency relationships, per the proposal's exclusions. This design says which package owns what, not what each `Depends` on.
- Choosing a status bar, launcher, or shell. The custom Quickshell shell is excluded, and this design does not pick a stand-in.

## Decisions

### One source package, four binary packages

All four packages are built from a single source package at `packages/gridline/`, which produces `gridline-session`, `gridline-config`, `gridline-tools`, and later `gridline-desktop`.

*Why:* The four binaries are developed, versioned, and released together. One source package gives them one version and one changelog, makes a file-conflict regression a build-time failure rather than an install-time surprise, and keeps `specs/package-delivery/spec.md`'s "no file is claimed twice" requirement cheap to verify.

*Alternative considered:* four source packages, one per binary. Rejected for now because it buys independent release cadence we do not need and costs a four-way version matrix plus cross-package dependency pinning on every change. The `packages/*/debian/` layout is kept precisely so additional source packages can be added later (a vendored compositor or shell would be a strong candidate) without moving anything.

*Decision confirmed:* one source package producing four binary packages is the chosen layout for the project. The `packages/*/debian/` structure exists so that additional source packages can be introduced later, not so that each binary gets its own source package.

### Source format 3.0 (native)

First-party Gridline packages use `3.0 (native)`: the upstream and the packaging are the same tree, so there is no upstream tarball to patch.

*Why:* Avoids maintaining a synthetic upstream/packaging split for code we author.

*Note on repository layout:* every Gridline source package lives under `packages/<name>/`, so `debian/` never appears at the repository root. The `.gitignore` ignores build staging artifacts by explicit path under `packages/*/debian/` and does not list `debian/source/` or `debian/patches/`, so both remain tracked. If a later source package vendors third-party code, it can adopt `3.0 (quilt)` and use `packages/<name>/debian/patches/` without a `.gitignore` change.

### Compositor adapter lives in gridline-session, behind an executable contract

The adapter is a set of compositor-specific executables plus one documented contract, shipped in `gridline-session`. The session launcher resolves which adapter to use, then speaks only the contract. Everything in `gridline-config` and `gridline-tools` is compositor-neutral.

*Why:* An executable boundary is verifiable by inspection, which is what `specs/compositor-abstraction/spec.md` asks for: "contains no compositor-specific command invocations, socket protocols, configuration formats, or process assumptions" is a grep-able claim about files outside one directory. A library boundary would be cheaper to call but easier to leak across.

*Why in `gridline-session`:* the adapter is integration glue, which is exactly what that package owns. Putting it in `gridline-tools` would force the diagnostic commands to carry compositor knowledge, contradicting the spec's requirement that common operations stay compositor-neutral.

*Alternative considered:* a `gridline-compositor-hyprland` binary package per compositor. Attractive once a second compositor exists, and the contract is designed so that extraction is mechanical. Not done now because a one-member plugin system is harder to justify than to add later.

*Boundary path:* the adapter layer is `packages/gridline/src/adapters/`, with one subdirectory per compositor, and `packages/gridline/src/adapters/hyprland/` as the only adapter in this phase. Everything outside `packages/gridline/src/adapters/` is compositor-neutral. This single path is what makes the isolation requirement in `specs/compositor-abstraction/spec.md` mechanically checkable rather than a matter of review judgment.

### Capability discovery is explicit and drives diagnostics

The adapter reports which required capabilities it can provide; the launcher checks that set before declaring the session up. A missing capability produces a structured, named condition that `gridline doctor` reads.

*Why:* This is what lets the compositor be replaceable in practice rather than on paper. It also gives the "launch failure returns control to the user" requirement in `specs/desktop-session/spec.md` something specific to report, instead of a crash with no attribution.

### Desktop identity is `Gridline:Hyprland`, most specific first

`XDG_CURRENT_DESKTOP` is set to a colon-separated list naming Gridline first and the active compositor second.

*Why:* This resolves a genuine conflict. `specs/desktop-session/spec.md` wants a stable Gridline identity; portal backends, including the Hyprland backend, select themselves by matching `XDG_CURRENT_DESKTOP`. A single value cannot do both: `Gridline` alone leaves the Hyprland portal backend unmatched, and `Hyprland` alone makes Gridline invisible to anything that wants to detect it. The list satisfies both, and the compositor component is supplied by the adapter rather than hardcoded, so it tracks whichever compositor is active.

*Trade-off accepted:* the second element changes if the compositor changes. The stable identity the spec requires is the first element, which is what Gridline-aware code should match on.

### Portal backend selection is configured, not raced

`gridline-config` ships an explicit portal configuration mapping each portal interface to a chosen backend, with the Hyprland backend taking screencast and related compositor-dependent interfaces and the GTK backend taking the rest.

*Why:* `specs/platform-integration/spec.md` requires determinism when two installed backends implement the same interface. Shipping the mapping makes the choice reviewable and diagnosable; relying on backend start order or implicit matching makes it neither.

### Gridline defaults are scoped to the Gridline session, never system-wide

`gridline-config` installs its defaults under a Gridline-specific directory beneath `/etc/xdg`, and the session launcher prepends that directory to `XDG_CONFIG_DIRS` **for the Gridline session only**. Nothing is written to the shared `/etc/xdg` search path that GNOME reads.

*Why:* This is the decision that protects the fallback guarantee. `/etc/xdg` is a shared search path: defaults dropped at its top level would be read by the GNOME session too, so a Gridline theme or GTK default could change the user's GNOME desktop, and removing Gridline could change it back. That would violate "GNOME survives Gridline install" and "no leftover Gridline state that alters its behavior" in `specs/desktop-session/spec.md`. Scoping the directory and injecting it per-session keeps the blast radius inside Gridline.

### Shipped defaults are dpkg conffiles

Defaults that an administrator is expected to edit are registered as conffiles.

*Why:* It is the mechanism that already implements "local changes survive upgrade" in `specs/package-delivery/spec.md`, including the prompt-on-conflict behavior. Reimplementing preservation in a maintainer script would be both more code and less predictable for administrators.

### Session locking is a required component, and its failure is fail-safe

Screen locking is treated as a required session component rather than an optional convenience, and a lock failure must never leave an unlocked, unattended session.

*Why:* `specs/platform-integration/spec.md` requires that a locked session survive suspend and that a lock component failure not expose the session. A lock that dies silently is the failure mode with real security consequences, so the launcher treats the lock component as required for capability discovery and reports its absence rather than continuing without it.

### AppArmor: ship profiles, never relax existing ones

Gridline may ship enforce-mode profiles for its own executables. No Gridline package modifies, disables, or sets any existing profile to complain mode.

*Why:* `specs/platform-integration/spec.md` requires it, and a desktop session is exactly the kind of component that is tempting to make work by loosening confinement. Stating the rule at design level makes the tempting fix an explicit spec violation rather than a judgment call under pressure.

## Risks / Trade-offs

- **Hyprland's supported interface surface may be narrower than Gridline needs** → The adapter's capability discovery turns this into a named, reported gap instead of a silent workaround. Gaps are recorded as upstream issues; `specs/compositor-abstraction/spec.md` already makes "requires an undocumented behavior" a review failure. Accept reduced functionality over an internal dependency.
- **Hyprland is a fast-moving upstream and may break its own documented interfaces** → Confining breakage to the adapter means a break is one package's bug, not a desktop-wide redesign. Capability discovery should fail loudly at session start rather than mid-session.
- **The dual `XDG_CURRENT_DESKTOP` value may confuse applications that expect a single token** → Needs explicit testing across GTK, Qt, and Snap-confined applications. If a major consumer mishandles the list, the fallback is to keep the list for portals and expose a separate Gridline-only variable for detection.
- **Snap confinement may not recognize Gridline as a known desktop** → Could affect theme access or portal routing for confined applications. The "confined graphical Snap runs" scenario is the canary; if it fails, the fix belongs upstream in the confinement interfaces rather than in loosened AppArmor.
- **Suspend and resume depend heavily on hardware and graphics drivers** → The baseline covers x86_64 broadly, but resume regressions tend to be device-specific, especially with hybrid or NVIDIA graphics. The design requires failures to be journal-attributable so a device-specific break is reportable rather than mysterious; per-device automation remains out of scope.
- **One source package means one release cadence** → A fix to `gridline-tools` ships a new version of all four. Acceptable while the packages move together; the per-source-package layout is the escape hatch.
- **The compositor abstraction is unvalidated with one implementation** → The boundary may encode Hyprland assumptions nobody notices until a second compositor arrives. Mitigated by specifying the contract in Gridline's own terms rather than by generalizing from Hyprland's interface, and by fixing the boundary at a single documented path, but this risk cannot be fully retired until a second adapter exists.
- **A future ISO is a strong temptation to put functionality in the image** → `specs/package-delivery/spec.md` makes image-exclusive capability a spec violation. The practical guard is that the image build consumes only published packages.

## Migration Plan

Not applicable in the usual sense: there is no existing Gridline to migrate from, and no user-facing deployment in this change, which produces specs only.

The rollback story for the eventual packages is the fallback guarantee itself: a user whose Gridline session misbehaves selects Ubuntu GNOME at the login screen, and an administrator removes the packages to return to a stock desktop. `specs/package-delivery/spec.md`'s clean-removal scenario is the test for that path, and it is worth treating as a release gate rather than an afterthought.

## Open Questions

- When `gridline-desktop` should start shipping, and whether it pulls the compositor itself or only the Gridline packages. Deferred: the boundary is specified, and the dependency question is out of scope here.
- Whether themes eventually warrant their own binary package, if theme data grows large relative to the rest of `gridline-config`. A later repackaging decision that does not affect these specs.
- Which component owns the ISO seed when the Remix work begins. Answerable later without changing the packages-only rule.
