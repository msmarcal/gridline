# Proposal: add-debian-package-skeleton

## Why

Gridline has 43 requirements and no file that `dpkg` can install. Every `add-*` change in the planned sequence assumes a package to ship into, and none of them creates one, which is why `define-debian-package-architecture` named this change as the owner of the buildable tree.

Three things change the moment `packages/gridline/debian/` exists, and all three are cheaper to get right before any content arrives:

- **Path ownership becomes real.** `package-delivery` says which package owns what. An `.install` manifest is what asserts it, so until one exists the boundary is a document rather than a fact `dpkg -L` can answer.
- **The build gate starts working.** The rule that no two packages ship the same path is only enforceable once there are packages built together to compare.
- **Verification stops being deferred.** The development host is Ubuntu 26.04.1 LTS resolute on amd64, which is exactly the declared baseline. Install, removal, upgrade and downgrade can be exercised for real rather than described.

## What Changes

**This change ships implementation.** It creates a buildable Debian source package and verifies it by building and installing on the supported baseline.

- Creates `packages/gridline/debian/`: `control` declaring three binary packages, `rules` with a debhelper sequence, `changelog`, `copyright` in machine-readable format 1.0 with the MIT text reproduced, and `source/format` set to `3.0 (native)`.
- Creates one `.install` manifest per binary package, declaring the Gridline-named directories each one owns. The packages ship those directories and no content; content arrives in the changes that own it.
- Adds a build-time gate that fails when two binary packages would ship the same path.
- Verifies on the baseline: the build produces exactly three `.deb` files, they install together without conflict, `dpkg -L` attributes each directory to the package the capability says owns it, removal returns the system to its prior state, and an upgrade followed by a downgrade leaves the system working.

Dependency fields are declared but carry no component yet. The packages depend on nothing because they contain nothing; the list accretes in the changes that use components, which is the policy `define-debian-package-architecture` settled.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

- `package-delivery`: gains two requirements. The first is that a Gridline package declares only Gridline-named paths. Inspecting the baseline showed why it is needed: `/usr/share/wayland-sessions`, `/usr/share/xdg-desktop-portal`, `/etc/xdg` and `/usr/libexec` are all already owned by archive packages. Directories are shared ownership in Debian, so declaring one produces no conflict and no error, which is exactly what makes it an easy mistake. The second is that removing a Gridline package leaves shared directories intact, which is the half that actually protects the fallback and the half that can be verified. No existing requirement is modified or removed.

## Impact

- **Affected specs:** `package-delivery`, by addition only.
- **Code:** creates `packages/gridline/debian/` and the build gate. First executable content in the repository.
- **Breaking changes:** none. Nothing was installable before.
- **System effects during implementation:** the verification tasks install and remove packages on the development host. They are empty packages owning four directories, and removal is itself one of the things being verified.
- **Unblocks:** every `add-*` change that needs a package to ship into, which is all of them.

## Out of scope

- The GDM session entry, the launcher, the compositor adapter. Those are `add-gdm-wayland-session` and `add-hyprland-session-baseline`, and they ship files into the directories this change creates.
- Shipped defaults, keybindings, themes, policy input, lock declarations and the generator. Those are `add-gridline-configuration-defaults`.
- Naming individual dependencies. The policy is settled; the entries follow from what later changes use.
- Maintainer scripts. Nothing in an empty package needs one, and adding a script with no work to do would be the kind of speculative machinery that is hard to remove later.
- `gridline-desktop`. The name stays reserved and this change must not produce it.
- CI. Re-running the gate and the install tests in a clean VM is `add-ci-vm-smoke-tests`; this change verifies them once, by hand, on the baseline.
- `lintian` policy cleanliness. It is not installed on the development host, and treating its output as a gate would change the host's package set to satisfy a check nobody has agreed to yet.

## Downstream changes

Every requirement this change adds or is assigned is the responsibility of exactly one change.

| Requirement | Owned by | Why there |
|---|---|---|
| A Gridline package declares only Gridline-named paths | `add-debian-package-skeleton` | The install manifests this change writes are where the declaration is made or avoided |
| Removing a Gridline package leaves shared directories intact | `add-debian-package-skeleton` | Removal behavior follows from what the manifests declare, and the baseline host can exercise it |
| Binary package names carry the gridline- prefix | `add-debian-package-skeleton` | Names are declared in the `control` this change creates |
| Dependencies are declared and satisfiable from the supported archive | `add-debian-package-skeleton` | The dependency fields exist here; their entries accrete later |
| Package ownership of a configuration artifact does not move once content ships | `add-debian-package-skeleton` | The `.install` manifests fix the boundary before any content exists to move |
| Upgrade and downgrade leave a working session and keep local configuration | `add-debian-package-skeleton` | The packaging that produces the behavior is created here, and the baseline host can exercise it |

`add-ci-vm-smoke-tests` owns no requirement here. It re-runs this change's verification in a clean VM, which is a different question from whether the behavior exists: this change establishes it on one machine, that change proves it on a machine nobody has touched.
