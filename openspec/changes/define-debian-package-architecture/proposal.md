# Proposal: define-debian-package-architecture

## Why

`package-delivery` settles what each Gridline package owns. It does not settle how those packages are built, how they declare what they need from the archive, what happens when one is upgraded, or how a file Gridline generates differs from a file an administrator is expected to edit.

Those are the questions the first implementation change will hit on its first day, and answering them in code rather than in specifications is how a packaging layout becomes accidental. Once `debian/control` exists, its boundaries are expensive to move: every `.install` file, every conffile registration and every dependency relationship encodes an assumption, and an administrator who has edited a shipped default is affected by every later correction.

Two inherited assignments also need a home before implementation starts. `session-manageability` requires policy input separate from what components read, and effective configuration produced by a Gridline-owned generator. Both were attributed to this change, because where policy input lives and where generated configuration comes from are package boundary decisions rather than configuration decisions.

This change ships documentation and specifications only. No `debian/` directory is created, no package is built, and no maintainer script is written.

## What Changes

- Adds the packaging rules that `package-delivery` leaves open: the binary package naming convention, dependency declaration policy, upgrade and downgrade behavior, the distinction between a conffile and a generated file, and the owning packages for policy input, generated configuration and lock declarations.
- Closes an exclusion the foundation declared deliberately. `establish-gridline-vision` put "declaring APT dependency relationships between packages" out of scope; this change takes it, because dependency declaration is a packaging architecture decision and nothing later can proceed without it.
- Records the source package layout, the build-time ownership gate and the source format as design decisions, since they are internal structure rather than observable behavior.
- Does **not** create the buildable tree. `packages/gridline/debian/` is delivered by `add-debian-package-skeleton`, which this change unblocks.

## Capabilities

### New Capabilities

None. The packaging rules this change adds belong to the capability that already owns the delivery contract; introducing a second packaging capability would split one concern across two specs.

### Modified Capabilities

- `package-delivery`: gains requirements for the binary package naming convention, dependency declarations satisfiable from the supported archive, upgrade and downgrade behavior, the conffile versus generated configuration distinction, and named owning packages for policy input, generated configuration and lock declarations. No existing requirement is modified or removed.

## Impact

- **Affected specs:** `package-delivery`, by addition only. `session-manageability` is not modified: its two assigned requirements are satisfied by the ownership requirement added here plus the design decisions below, not by changing their text.
- **Code:** none. This change writes documentation and specifications.
- **Breaking changes:** none. Nothing ships yet.
- **Unblocks:** `add-debian-package-skeleton`, which creates `packages/gridline/debian/` against the rules defined here, and through it every `add-*` change that needs a package to ship into.

## Out of scope

- Creating `debian/control`, `debian/rules`, `debian/changelog`, `.install` files or maintainer scripts. Those are `add-debian-package-skeleton`.
- The contents of defaults, keybindings and themes, which remain `add-gridline-configuration-defaults`.
- The policy input format and the generator's implementation. This change says which package owns them; `add-gridline-configuration-defaults` decides what they look like.
- Naming specific archive packages as dependencies. This change sets the policy that dependencies must be declared and satisfiable from the supported archive; which ones they are follows from what each later change actually uses.
- CI and build infrastructure, which is `add-ci-vm-smoke-tests`.

## Downstream changes

Every requirement added here is the responsibility of exactly one of the changes below.

| Requirement | Owned by | Why there |
|---|---|---|
| Binary package names carry the gridline- prefix | `add-debian-package-skeleton` | Binary package names are declared in `debian/control`, which that change creates |
| Dependencies are declared and satisfiable from the supported archive | `add-debian-package-skeleton` | Dependency fields live in `debian/control`. The list accretes as later changes use components, but the declarations and their archive-component annotations are that change's structure |
| Upgrade and downgrade leave a working session and keep local configuration | `add-debian-package-skeleton` | Conffile registration and maintainer scripts are what preserve an administrator's edit across an upgrade. `add-ci-vm-smoke-tests` verifies the behavior but does not deliver it |
| Generated configuration is distinguishable from shipped editable defaults | `add-gridline-configuration-defaults` | The requirement is about files that do not exist until that change ships policy input and generated output. All three of its scenarios are vacuous in the skeleton, which registers a destination and generates nothing |
| Policy input, generated configuration and lock declarations have named owning packages | `add-debian-package-skeleton` | Path ownership is asserted by `.install` manifests, so the boundary becomes real when they are written |

Two attributions are worth their reasoning, because each could plausibly sit elsewhere.

**Upgrade and downgrade** is owned by the change that delivers the guarantee rather than the change that proves it, following the precedent already set for recoverability in `desktop-session`: that requirement is owned by `add-gdm-wayland-session`, which delivers the fallback, while `add-session-health-check` only makes the failure diagnosable. `add-ci-vm-smoke-tests` stands in the same relation here.

**Generated configuration** is owned by `add-gridline-configuration-defaults` rather than by the skeleton, because a requirement about generated files cannot be satisfied before generated files exist. The skeleton carries a precondition instead: it must not register the generated destination as a conffile, so that the later change inherits a registration that can express the distinction.

`add-ci-vm-smoke-tests` owns no requirement added here, so it does not appear in the table. Its role is verification: upgrade, downgrade and clean removal on the supported baseline are behaviors it exercises rather than delivers.

`add-debian-package-skeleton` is new, and it exists because the foundation's sequence named no owner for the buildable tree. Every `add-*` change in that sequence assumes a package to ship into, and none of them creates one. Splitting it out keeps this change inside the project's rule that a definition change produces documentation and specifications only.
