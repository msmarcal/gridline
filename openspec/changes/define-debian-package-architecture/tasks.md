# Tasks

Scope note: this change produces documentation and specifications only. No `debian/` directory is created, no package is built, no maintainer script is written, and no dependency is named. `packages/gridline/debian/` belongs to `add-debian-package-skeleton`, which this change unblocks. Where a rule here constrains a later change, the task records the rule rather than applying it.

## 1. Packaging architecture document

- [ ] 1.1 Write `docs/packaging.md` recording the source package layout, the `3.0 (native)` source format, and the one-source-package decision, and verify it does not contradict `packages/gridline/README.md`, which already states the same layout
- [ ] 1.2 Record in `docs/packaging.md` the build-time file-ownership gate, naming which change implements it and which re-runs it in CI, and verify the named changes match the downstream table in `proposal.md`
- [ ] 1.3 Record in `docs/packaging.md` the binary package naming convention and its dependence on the `.gitignore` staging glob, and verify the stated glob matches the one in `.gitignore`

## 2. Conffile and generated configuration rule

- [ ] 2.1 Record in `docs/packaging.md` the authorship test that decides conffile versus generated, as a table covering shipped defaults, policy input, lock declarations and generated configuration, and verify every one of the four has a stated registration and a reason
- [ ] 2.2 Verify the rule in `docs/packaging.md` does not contradict the conffile requirement in `openspec/specs/package-delivery/spec.md` or the generated-configuration statement in `docs/configuration.md`
- [ ] 2.3 Record in `docs/configuration.md` a cross-reference to the authorship rule, and verify the two documents state the same rule rather than two similar ones

## 3. Ownership of policy artifacts

- [ ] 3.1 Record in `docs/packaging.md` and `docs/package-matrix.md` the owning package for policy input, lock declarations and the generated configuration destination, and verify each of the three names exactly one package in both documents
- [ ] 3.2 Record that the generator executable is owned by `gridline-tools` while the output path is owned by `gridline-config`, with the reason, and verify the statement is consistent with the ownership rows in `openspec/specs/package-delivery/spec.md`
- [ ] 3.3 Verify the recorded ownership satisfies the two `session-manageability` requirements assigned to this change, by checking each against `openspec/specs/session-manageability/spec.md` and noting which requirement each ownership statement serves

## 4. Dependency policy

- [ ] 4.1 Record in `docs/packaging.md` the rule that a component whose absence blocks session startup is a dependency and one whose absence degrades is a recommendation, and verify the blocking and degrading classifications cited match the component table in `docs/architecture.md`
- [ ] 4.2 Record the rule that each dependency's archive component is declared alongside it, with the support consequence of universe stated, and verify the statement matches the universe gap already recorded in `docs/security-and-manageability.md`
- [ ] 4.3 Verify `docs/packaging.md` names no individual dependency, since which components are required follows from later changes rather than from this one

## 5. Upgrade, downgrade and removal

- [ ] 5.1 Record in `docs/packaging.md` the expected behavior for upgrade, downgrade and interrupted upgrade, and verify each of the three has a stated outcome rather than only a goal
- [ ] 5.2 Verify the recorded behavior matches the upgrade and downgrade requirements added in `specs/package-delivery/spec.md`, and that removal of generated files is consistent with the clean-removal requirement already in `openspec/specs/package-delivery/spec.md`

## 6. Sequence and consistency

- [ ] 6.1 Add `add-debian-package-skeleton` to the recorded downstream sequence in `docs/architecture.md`, with what it delivers, and verify the change set named there matches the downstream table in this change's `proposal.md`
- [ ] 6.2 Verify every requirement added in `specs/package-delivery/spec.md` appears in this change's downstream table with exactly one owning change, and that the set of changes named in the table is the set named in the sequence recorded in `docs/architecture.md`
- [ ] 6.3 Verify no file under `packages/gridline/debian/` was created by this change, and that `docs/packaging.md` states the skeleton is delivered elsewhere

## Workflow follow-up

- Run `/opsx:verify` once the documents above exist, to check them against the specs in this change.
- Archive this change, then propose `add-debian-package-skeleton`, which is the first change that produces a buildable package.
