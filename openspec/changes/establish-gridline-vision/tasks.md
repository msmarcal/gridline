# Tasks

Scope note: this change produces documentation and specifications only. No package is built, no behavior is implemented, and no hardware validation is performed here. Implementation and verification tasks belong to the downstream changes named in `proposal.md`. Where a requirement is satisfied by Ubuntu's existing stack, the downstream work is to keep it working and prove it, not to reimplement it.

## 1. Project foundation documents

- [ ] 1.1 Write `docs/vision.md` recording the product statement, the supported baseline of Ubuntu 26.04 LTS on x86_64, the additive and removable session model, and the project-wide non-goals, and verify it states explicitly that GNOME remains a supported fallback
- [ ] 1.2 Write `docs/architecture.md` describing the session, configuration, tools, and compositor adapter layers and the boundaries between them, and verify every capability named in `proposal.md` maps to exactly one documented layer
- [ ] 1.3 Write `docs/component-ownership.md` with one row per component covering upstream, Ubuntu package source, project ownership, and fallback behavior, and verify no component is listed without a named owner and a fallback
- [ ] 1.4 Write `docs/package-matrix.md` listing each planned binary package, what it owns, and which capability requires it, and verify the ownership statements match `specs/package-delivery/spec.md` with no contradiction

## 2. Structural decisions recorded

- [ ] 2.1 Document the repository layout and the one-source-package decision in `packages/gridline/README.md`, including room for future source packages, and verify the documented layout matches what `.gitignore` already anticipates
- [ ] 2.2 Record in `packages/gridline/README.md` that `gridline-desktop` is a reserved metapackage boundary with no implementation in this change, and verify the statement matches the reserved requirement in `specs/package-delivery/spec.md`
- [ ] 2.3 Document the compositor adapter contract in `docs/compositor-adapter.md`, including the capability set an adapter must report and the rule that only documented compositor interfaces may be used, and verify the document defines the contract without specifying a Hyprland implementation
- [ ] 2.4 Document the adapter extraction path for a future second compositor in `docs/compositor-adapter.md`, and verify it identifies which parts of the desktop would and would not need to change
- [ ] 2.5 Document configuration precedence and the session-scoped `XDG_CONFIG_DIRS` rule in `docs/configuration.md`, and verify it states that Gridline defaults never alter configuration resolution for a GNOME session

## 3. Recovery and fallback documentation

- [ ] 3.1 Document the user recovery path for a failed Gridline session in `docs/vision.md` or a dedicated `docs/recovery.md`, covering session selection at GDM and where the failure is recorded, and verify the path requires no working Gridline desktop to follow
- [ ] 3.2 Verify the documented recovery path matches the fallback requirements in `specs/desktop-session/spec.md` with no contradiction

## 4. Change sequencing

- [ ] 4.1 Record the planned downstream change sequence and what each one delivers in `docs/architecture.md` or `docs/vision.md`, and verify every implementation task removed from this change is attributed to a named downstream change

## Workflow follow-up

- Run `/opsx:verify` once the documents above exist, to check them against the specs in this change.
- Archive the change after the project's review requirements are satisfied, then open `define-debian-package-architecture`.
