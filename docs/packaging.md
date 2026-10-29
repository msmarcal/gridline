# Packaging architecture

How Gridline's packages are built and how they behave across install, upgrade and removal.

`openspec/specs/package-delivery/spec.md` fixes what each package owns and how it must behave. This page records the structure and the rules that produce that behavior. Where the two disagree, the spec wins.

**The buildable tree is not here.** `packages/gridline/debian/` is delivered by `add-debian-package-skeleton`. This document is what that change builds against.

## Source package layout

One source package at `packages/gridline/` produces every binary package: `gridline-session`, `gridline-config` and `gridline-tools`, with `gridline-desktop` reserved for later.

The binaries are developed, versioned and released together, so a single source package gives them one version and one changelog. It also turns a file-ownership collision into a build failure rather than an install-time conflict, which is what makes the ownership gate below possible at all: a gate only sees the packages built alongside it.

The cost is one release cadence. A fix to `gridline-tools` ships a new version of all of them. That is accepted while they move together, and the `packages/<name>/` layout leaves room for a genuinely separate source package later, such as one that vendors third-party code.

This restates the layout already recorded in [packages/gridline/README.md](../packages/gridline/README.md) rather than introducing it. The two documents must agree; that README is the one a packager reads first.

## Source format

`3.0 (native)`. The upstream and the packaging are the same tree, so there is no upstream tarball to patch and no synthetic upstream split to maintain for code the project authors.

`debian/patches/` stays tracked and unused. It is available if a later source package vendors third-party code and adopts `3.0 (quilt)`.

## The gridline- prefix

Every binary package is named `gridline-<something>`. That is a requirement, because package names are what a user types and what APT resolves.

It also does work inside the repository. The `.gitignore` carries:

```
packages/*/debian/gridline-*/
```

One glob covers every binary package's build staging directory, present and future. **Adding a binary package therefore needs no new ignore rule**, which is the failure mode the convention exists to prevent: a package added without its staging directory ignored leaves untracked files after every build.

That half is not a requirement. A version-control rule is not behavior of the shipped software, and nobody installing or administering Gridline can observe it. If a future package genuinely cannot carry the prefix, the glob has to be revisited in the same change.

## File ownership is gated at build time

**The build fails when two binary packages would ship the same path.**

`package-delivery` requires that no file be claimed twice and that installation complete without conflicts. A build-time gate turns that from something a tester might notice into something that cannot be committed.

| Role | Change |
|---|---|
| Implements the gate in the build | `add-debian-package-skeleton` |
| Re-runs the gate in CI | `add-ci-vm-smoke-tests` |

Those are the changes named in the attribution table in this change's `proposal.md`. This document specifies that the gate exists, not how it is written.

## Conffile or generated: the test is authorship

The question is not where a file lives but who wrote it.

| File | Registration | Why |
|---|---|---|
| Shipped default an administrator is expected to edit | dpkg conffile | Preserving the edit and prompting on conflict is exactly what dpkg already does |
| Policy input | dpkg conffile | Authored by an administrator; being edited is its entire purpose |
| Lock declarations | dpkg conffile | Authored by an administrator, alongside the policy they lock |
| Generated configuration | **Not a conffile**, and removed when its owning package is removed | Produced from the two above. A preserved edit would be discarded by the next regeneration, so inviting one is worse than refusing it |

A conffile prompt asks an administrator to protect their edit. Asking that about a file the next generation overwrites teaches them the prompt is noise, and the next prompt that matters gets the same reflex.

The trade-off is accepted rather than solved: an administrator who edits generated output loses the edit at the next regeneration. The mitigation is that generated files identify themselves as generated and name the input to change instead, which is cheaper and more honest than attempting a merge.

This rule is the same one stated in [configuration.md](configuration.md), from the configuration side. Neither document may state it differently from the other.

## Declared paths, and the ones deliberately not declared

A package declares what it owns in its `debian/<package>.dirs` and `debian/<package>.install` manifests. These are the paths Gridline claims:

| Package | Declares |
|---|---|
| `gridline-session` | `/usr/libexec/gridline/` |
| `gridline-config` | `/etc/xdg/gridline/`, `/etc/xdg/gridline/generated/` |
| `gridline-tools` | `/usr/share/gridline/` |

**No shared system directory is declared.** `/usr/share/wayland-sessions`, `/usr/share/xdg-desktop-portal`, `/usr/lib/systemd/user`, `/etc/xdg` and `/usr/libexec` all belong to archive packages already, and Gridline declares none of them. Files land in them later as Gridline-named drop-ins; the directories stay theirs.

The reason is not conflict avoidance, because there is no conflict to avoid: a directory in Debian is shared ownership, so declaring one installs cleanly and reports nothing. That is what makes it an easy mistake. The reason is the fallback guarantee. A Gridline package that declared `/etc/xdg` would be a Gridline package whose removal could touch a directory the stock session reads.

**`dpkg` still records those shared parents against Gridline, and that is expected.** `dpkg -S /etc/xdg` lists `gridline-config` alongside `systemd`, `snapd` and a dozen others, because a package containing `/etc/xdg/gridline/` necessarily contains `/etc/xdg/` too. Every package that installs under a shared directory appears the same way. Declared ownership is what the manifests claim; the `dpkg` directory list is the transitive closure of that, and the two are not the same question.

What is verifiable, and verified on the baseline: removing every Gridline package leaves the shared directories in place with their archive owners unchanged, and takes every Gridline-named directory with it.

## Ownership of configuration artifacts

Each artifact is owned by exactly one package, and both ownership lists are closed by an exclusivity clause in `openspec/specs/package-delivery/spec.md`.

| Artifact | Owning package |
|---|---|
| Shipped defaults, keybindings, themes | `gridline-config` |
| Administrator's policy input | `gridline-config` |
| Lock declarations | `gridline-config` |
| Destination of the generated configuration | `gridline-config` |
| The generator that produces the effective configuration | `gridline-tools` |

**Policy input belongs with the defaults** because policy input is a default an administrator sets. Putting it in `gridline-session` would move configuration content into the integration package and break the boundary `package-delivery` fixes.

**The generator belongs with the commands**, which is what `gridline-tools` owns.

So the output path and the program that writes it are owned by different packages, deliberately. The alternative, moving the generator into `gridline-config`, would put an executable in the package that exists to ship data.

Two consequences the skeleton has to respect:

- `gridline-config` ships the destination and no generated content. Installed before anything has been generated, it owns an empty destination.
- Removing `gridline-tools` leaves `gridline-config` owning a destination that nothing can regenerate. The destination stays owned and no orphaned generated file is left behind. This must not be expressed as a circular dependency between the two packages.

### Why the generator runs at runtime

Policy input is authored by an administrator, so its values do not exist at build time. The package ships the input and the destination; the generator runs on the running system when policy changes.

A build-time generator could only generate from the shipped defaults, which is precisely the case that needs no generator.

## Licence and copyright

The project is MIT, recorded in [LICENSE](../LICENSE). The only condition is
attribution: the copyright notice and the licence text travel with copies and
with substantial portions.

`debian/copyright` is mandatory, and `add-debian-package-skeleton` writes it in
machine-readable format 1.0. Three consequences for that change:

- The `Files: *` stanza is `MIT`, with the full licence text reproduced in the
  stanza rather than referenced from `/usr/share/common-licenses/`, which
  carries the GPL and LGPL texts but not MIT.
- Gridline links nothing and vendors nothing today, so no second stanza is
  needed. A source package that later vendors third-party code gets its own
  stanza, and that is one of the two reasons the layout leaves room for a
  separate source package.
- MIT imposes no obligation on anyone depending on a Gridline package, so the
  licence does not constrain the dependency policy below. The archive component
  of a dependency matters for support; its licence does not matter for ours.

Gridline configures archive packages and ships no modified copy of any of them,
so their licences stay theirs and are not restated here.

## Dependency policy

**A component whose absence blocks session startup is a dependency. A component whose absence degrades the session is a recommendation.**

The strength is not a packager's judgment call: [architecture.md](architecture.md) already classifies every component as blocking startup or degrading, and the declaration follows that classification. Declaring a blocking component as a recommendation would let APT produce an installation that cannot start a session at all.

Every dependency resolves on the supported baseline using the Ubuntu archive alone. No Gridline package requires a third-party repository to install.

**Each dependency records whether it comes from main or from universe.** That is a support fact rather than packaging trivia. Universe carries **no Canonical security-update commitment**. Main does. Patching a universe component is therefore the operator's responsibility, which on a personal machine means its owner. The gap is recorded in [security-and-manageability.md](security-and-manageability.md); a per-dependency annotation is what makes the total visible without a separate audit.

**This document names no individual dependency.** Which components Gridline requires follows from what later changes actually use, so the list accretes in `add-debian-package-skeleton` and after. Settling the policy before the list is the point: an early mistake then is a missing entry rather than a wrong structure.

## Upgrade, downgrade and interrupted upgrade

| Event | Outcome |
|---|---|
| **Upgrade** | Administrator-modified configuration is preserved, and the session starts normally afterwards. Preservation comes from conffile registration, which is why the authorship test above decides it |
| **Downgrade** | The session starts normally on the older version, and no file from the newer version is left behind that changes its behavior |
| **Interrupted upgrade** | Completing the package configuration reaches the same state as an uninterrupted upgrade. No manual file repair is required at any point |

Generated configuration is removed with its owning package, which is consistent with the clean-removal requirement already in `openspec/specs/package-delivery/spec.md`: removal returns the system to a stock desktop, leaving nothing behind except what an administrator authored. Generated output is not authored, so nothing is lost by removing it.

These behaviors become testable when `add-debian-package-skeleton` produces packages, and verified when `add-ci-vm-smoke-tests` exercises them on the supported baseline.

## Related documents

- [packages/gridline/README.md](../packages/gridline/README.md): the source package itself, and the build command
- [package-matrix.md](package-matrix.md): what each binary package owns
- [configuration.md](configuration.md): configuration precedence, and the authorship rule from the configuration side
- [architecture.md](architecture.md): the blocking versus degrading classification that dependency strength follows
- [security-and-manageability.md](security-and-manageability.md): the universe security-update gap
- [LICENSE](../LICENSE): the MIT text that `debian/copyright` reproduces
