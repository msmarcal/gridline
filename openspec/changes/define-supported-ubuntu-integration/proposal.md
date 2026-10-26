# Proposal

## Why

`platform-integration` carries twelve requirements describing what must work inside a running Gridline session: networking, audio, Bluetooth, portals, Snaps, power, locking, polkit, XWayland, AppArmor, and the update path. Not one of them can be exercised today, because there is no compositor and therefore no session to run them in. They are not wrong; they are unreachable, and they stay unreachable until `add-hyprland-session-baseline` lands.

What is reachable now is the layer underneath them: which Ubuntu packages those twelve requirements rest on, whether each exists on the supported baseline, which archive component provides it, and whether its absence blocks the session or degrades it. That layer is currently belief rather than record.

`docs/component-ownership.md` lists every component with an archive component and a fallback, and `docs/package-matrix.md` records versions with a note saying that confirming them against the archive belongs to this change. Both are honest accounts of what someone believed when writing them. Neither is a requirement, so nothing fails when the archive moves underneath them, and nothing says the two documents must agree.

The project's own context states the standard this falls short of: on a released baseline, package availability, version and archive component are facts to verify against the archive rather than assumptions to carry.

## What Changes

**This change produces documentation and specs only.** It builds no package, edits no packaging metadata, and runs no hardware validation. It establishes what the component set is and makes the record binding; the change that can act on a component is the one that declares it.

- Verifies every component the twelve `platform-integration` requirements rest on against the baseline archive: existence, version, and archive component. The measurement is recorded in design.md: all nineteen exist, fourteen in `main`, five in `universe`, and every `universe` entry is Hypr-ecosystem.
- Classifies each component by what its absence does, blocking session startup or degrading a capability within it, and records that classification where the component is described rather than leaving it implied by prose.
- Records which blocking components cannot be declared yet and which change owns each, so a deferred dependency is attributed rather than merely missing.
- Reconciles `docs/component-ownership.md` and `docs/package-matrix.md` against the measurement, states which document is authoritative where they overlap, and corrects the polkit daemon's package name, which this baseline provides as `polkitd`.
- Adds requirements making the component set a declared fact, making each component's archive support status part of its record, and making a drifted record a detectable defect.

**Out of scope.** No dependency field is written into `debian/control`; that is implementation, and it belongs to the change that delivers a session able to use the component. No integration behavior is proved: no audio is played, no device is paired, no Snap is launched. The compositor, the compositor's portal backend, the screen locker and the polkit agent are not selected or declared here. No AppArmor profile is authored, and the portal configuration file's content stays with `add-desktop-portal-integration`.

## Capabilities

### New Capabilities

None. `platform-integration` is the cohesive owner of the integration surface, and the component set is part of that surface rather than a separate concern.

### Modified Capabilities

- `platform-integration`: gains requirements that required components are declared with the behavior when each is absent, that each component's archive support status is recorded where it is relied on, and that recorded component facts are verifiable against the supported archive. No existing requirement changes.

`package-delivery` is deliberately not modified. Its requirement that dependencies be declared and satisfiable from the supported archive already says what a later change will implement; the gap there is implementation, not specification, and inventing a requirement to make this change look larger would be wrong.

## Impact

- `docs/component-ownership.md` and `docs/package-matrix.md`: reconciled with the measured archive, with authority between them stated.
- `openspec/specs/platform-integration/spec.md`: three new requirements, via this change's delta.
- `packages/`: untouched. No control file, no build, no installed path changes.
- Two findings are recorded for the changes that own them rather than fixed here: `debian/control` still describes `gridline-session` as shipping no content, which stopped being true when `add-gdm-wayland-session` was archived, and all three packages still declare `Depends: ${misc:Depends}` and nothing else.
