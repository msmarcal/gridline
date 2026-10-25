# Design

## Context

See proposal.md - Why. `openspec/specs/package-delivery/spec.md` already fixes what each package owns; this change settles how the packages are built and how they behave across installs and upgrades.

Constraints that shape the decisions below:

- **This change writes no `debian/` directory.** It is a definition change, so every decision here is a commitment the next change implements, not a file this one creates.
- **The archive owns its own files.** Gridline influences a third-party component only through a Gridline-named drop-in, never by editing a file another package owns.
- **Most of the Hypr stack is in universe**, which carries no Canonical security-update commitment. Dependency decisions are support decisions, not only technical ones.
- **A generator exists.** `session-manageability` requires effective configuration to be produced by Gridline rather than resolved by a search path, which puts a build-time and runtime output into a packaging layout that otherwise only ships static files.

## Goals / Non-Goals

**Goals:**

- A source layout and build gate that make the "no file is claimed twice" requirement fail at build time rather than at install time.
- A rule for conffile versus generated file that an implementer can apply without judgment per file.
- Named owning packages for policy input, generated configuration and lock declarations.
- A dependency policy that keeps the session installable from the archive alone and makes the universe exposure visible.

**Non-Goals:**

- Naming individual dependencies. The policy is settled here; the list follows from what later changes use.
- The policy input format, the generator's language, or its invocation mechanism.
- Build infrastructure and CI.

## Decisions

### One source package at `packages/gridline/`, producing every binary

`packages/gridline/` produces `gridline-session`, `gridline-config` and `gridline-tools`, with `gridline-desktop` reserved.

*Why:* The binaries move together, so one source package gives them one version and one changelog, and it makes a file-ownership collision a build failure instead of an install-time conflict. This confirms the layout already recorded in `packages/gridline/README.md` rather than introducing it.

*Alternative considered:* one source package per binary. Rejected: it buys independent release cadence the project does not need and costs a version matrix plus cross-package pinning on every change. The `packages/<name>/` layout keeps the option open for a genuinely separate source package later, such as one that vendors third-party code.

### The `gridline-` prefix is what keeps build staging out of version control

The repository ignores `packages/*/debian/gridline-*/`, so every binary package's build staging directory is excluded by the prefix convention alone. Adding a binary package therefore needs no new ignore rule.

*Why this is a design decision and not a requirement:* a version-control rule is not behavior of the shipped software. Nobody installing or administering Gridline can observe it, so it belongs here and in `docs/packaging.md` rather than in a scenario. The prefix requirement in the spec states the observable half, that every binary package name begins with `gridline-`.

### Source format `3.0 (native)`

The upstream and the packaging are the same tree, so there is no upstream tarball to patch.

*Why:* Avoids maintaining a synthetic upstream split for code the project authors. `debian/patches/` stays tracked and unused, available if a later source package vendors third-party code as `3.0 (quilt)`.

### File ownership is gated at build time

The build fails when two binary packages would ship the same path.

*Why:* `package-delivery` requires that no file be claimed twice and that installation complete without conflicts. A gate at build time turns that from a property someone notices during a test install into one that cannot be committed. It is also why one source package is worth more than independent cadence: the gate only sees packages built together.

*Where it runs:* in the build, owned by `add-debian-package-skeleton`, and re-run in CI by `add-ci-vm-smoke-tests`. This change specifies that it exists, not how it is written.

### Conffile when an administrator authors it, generated when Gridline produces it

The test is authorship, not location:

| File | Registration | Why |
|---|---|---|
| Shipped default an administrator is expected to edit | dpkg conffile | Preserving the edit and prompting on conflict is exactly dpkg's behavior |
| Policy input | dpkg conffile | Authored by an administrator; its whole purpose is to be edited |
| Lock declarations | dpkg conffile | Authored by an administrator, alongside the policy they lock |
| Generated configuration | Not a conffile; removed on package removal | Produced from the two above; a preserved edit would be discarded by the next regeneration, so inviting one is worse than refusing it |

*Why:* A conffile prompt asks the administrator to protect their edit. Asking that about a file the next generation overwrites teaches them the prompt is noise. The rule also gives `add-gridline-configuration-defaults` a mechanical answer for each file it adds.

*Trade-off accepted:* an administrator who edits generated output loses the edit silently at regeneration. Mitigated by the requirement that generated files identify themselves and name the supported input, which is cheaper and more honest than trying to merge.

### Policy input and lock declarations go in `gridline-config`; generated configuration is owned by `gridline-config` and produced at runtime

`gridline-config` owns all three paths: policy input, lock declarations, and the generated output's location.

*Why `gridline-config`:* it already owns the shipped defaults, and policy input is a default an administrator sets. Splitting policy into `gridline-session` would put configuration content in the integration package and break the boundary `package-delivery` fixes.

*Why generated at runtime rather than at build:* policy input is administrator-authored, so its values are not known at build time. The package ships the input and the destination; the generator runs on the running system when policy changes. A build-time generator would only be able to generate from the shipped defaults, which is the case that needs no generator.

*Consequence for the skeleton:* `gridline-config` ships a destination directory and no generated content, and the generator executable itself belongs to `gridline-tools`, which owns Gridline's commands. Ownership of the output path and ownership of the program that writes it are deliberately different packages.

### Dependencies are declared strictly, with the archive component recorded

A component whose absence blocks session startup is a `Depends`. A component whose absence degrades the session is a `Recommends`. The archive component of each is recorded alongside the declaration.

*Why strict:* `architecture.md` already classifies every component as blocking or degrading, so the dependency strength follows from a decision already made rather than from packager taste. Declaring a blocking component as a recommendation would let APT produce an installation that cannot start a session.

*Why record main versus universe:* the universe exposure is the project's largest gap between promise and archive, and a dependency list is where it becomes concrete. Recording it per dependency makes the total visible without a separate audit.

## Risks / Trade-offs

- **The generator's output path is owned by one package and written by another** → A removal of `gridline-tools` leaves `gridline-config` owning a destination nothing can regenerate. Acceptable because the output is removable and the session reports policy that is not in effect, but `add-debian-package-skeleton` should not express this as a circular dependency.
- **Strict `Depends` on universe packages ties a main-quality promise to universe-quality support** → No packaging choice fixes this; the honest handling is the recorded component plus the documented gap. A later flavor candidacy would have to move components, not re-declare them.
- **Declaring dependencies now, before the changes that use the components** → The policy is settled here and the list accretes later, so an early mistake is a missing entry rather than a wrong structure. The risk is a dependency nobody adds; the build gate does not catch that, only a clean-VM install does, which is why `add-ci-vm-smoke-tests` carries it.
- **One source package means one release cadence** → A fix to `gridline-tools` ships a new version of all of them. Accepted while they move together; a separate source package is the escape hatch and the layout already allows it.
- **The conffile rule depends on classifying each file correctly** → A file misclassified as generated loses administrator edits. The rule is authorship-based to make misclassification visible in review, but it is a rule applied by people, and `add-gridline-configuration-defaults` is where it will be applied most.

## Migration Plan

Not applicable. Nothing is installed, so there is nothing to migrate. The upgrade and downgrade requirements added here describe behavior that first becomes testable when `add-debian-package-skeleton` produces packages, and `add-ci-vm-smoke-tests` is where they become verified rather than specified.

## Open Questions

- Whether `gridline-desktop`, when it ships, depends on the compositor itself or only on the Gridline packages. Deferred: it changes nothing about the boundaries defined here.
- Whether themes eventually warrant a separate binary package if theme data grows relative to the rest of `gridline-config`. A repackaging decision that does not affect these requirements.
