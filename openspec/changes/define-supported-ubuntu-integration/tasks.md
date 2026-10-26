# Tasks

This change produces documentation and specs only. No task builds a package, edits
packaging metadata, or validates hardware. A task that finds packaging work to do
records it for the change that owns it rather than doing it.

## 1. Confirm the measurement

- [x] 1.1 Re-run the availability query against a system on the supported baseline for every component design.md tabulates, and verify each one's existence, version and archive component against the table there, recording any that moved since 2026-10-26 rather than silently accepting the design's values
- [x] 1.2 Verify the query used is one a reader can repeat with stock tools and no Gridline package installed, and record the exact command and the date in `docs/package-matrix.md`
- [x] 1.3 Verify that every component named in `docs/component-ownership.md` appears in the measurement, and that the measurement introduces no component the ownership matrix does not carry; report either direction of mismatch as a finding rather than adding rows to make the two agree

## 2. Record the classification

- [ ] 2.1 Record in `docs/component-ownership.md`, for every component, whether its absence blocks session startup or degrades a named capability, and verify no row states only that the component is used
- [ ] 2.2 Verify the blocking set recorded matches the five components `docs/architecture.md` names as blocking, and that any difference is resolved by correcting one document rather than leaving both
- [ ] 2.3 Verify the degrading set is exactly the nine components design.md lists, and that each row names which capability degrades rather than saying the session degrades generally
- [ ] 2.4 Verify every row that names "Not applicable" as its fallback states why the component cannot be absent without the baseline itself being broken, so the category stays deliberate rather than becoming a blank

## 3. Attribute the deferred declarations

- [ ] 3.1 Record in `docs/component-ownership.md`, for each blocking component no Gridline package can declare yet, the change that owns the declaration, and verify each named change exists in the downstream sequence table in `docs/architecture.md`
- [ ] 3.2 Verify the compositor adapter is recorded as Gridline's own rather than as a deferred third-party declaration, since it ships in `gridline-session` already
- [ ] 3.3 Record that all three binary packages currently declare `Depends: ${misc:Depends}` and nothing else, and verify this is stated as a known gap attributed to a later change rather than as the intended end state

## 4. Record archive support status

- [ ] 4.1 Verify every component row in `docs/component-ownership.md` names its archive component, and that the five `universe` entries are marked distinctly enough that a reader scanning the table sees them without reading prose
- [ ] 4.2 Verify each `universe` row states, alongside the dependency, that no Canonical security-update commitment covers it, and that the statement is the same claim in the same words as `docs/security-and-manageability.md` makes, so the two copies cannot drift
- [ ] 4.3 Correct the polkit component's name in `docs/component-ownership.md`, which carries `polkit-1`, a name that is neither the binary package nor the source package on this baseline. Verify the replacement by the column it sits in: the Ubuntu source column takes `policykit-1`, and `polkitd` appears only where the binary is what matters, such as a statement about what is installed. Verify by grep that no live document still carries `polkit-1`, and that archived artifacts keep theirs unchanged

## 5. Reconcile the two documents

- [ ] 5.1 Update the archive table in `docs/package-matrix.md` to match the confirmed measurement, and verify every version and component in it appears identically in `docs/component-ownership.md`
- [ ] 5.2 State in both documents that `docs/component-ownership.md` is authoritative for component facts where the two overlap, and verify the statement appears in both rather than only in the one that wins
- [ ] 5.3 Remove or update the note in `docs/package-matrix.md` saying confirmation against the archive is owned by this change, and verify what replaces it names who owns re-checking from now on

## 6. Coherence

- [ ] 6.1 Verify each of the three requirements added by this change's delta is satisfied by something a reader can check in the documents, and name for each requirement the document and section that satisfies it
- [ ] 6.2 Verify no task in this change edited anything under `packages/`, by confirming the working tree shows no change to that directory
- [ ] 6.3 Verify the findings recorded for later changes, the stale `gridline-session` description and the absent dependency fields, each name an owning change, and that neither was fixed here

## Workflow follow-up

- Run `/opsx:verify` once the documents and the delta are complete.
- Archive the change, then propose `add-hyprland-session-baseline`, which delivers the compositor adapter and is the first change able to declare a blocking dependency this change recorded.
