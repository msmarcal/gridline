# Tasks

Scope note: this change produces documentation and specifications only. No policy format is fixed, no generator is implemented, and no package is built. Implementation belongs to `define-debian-package-architecture` and `add-gridline-configuration-defaults`, which this change unblocks.

## 1. Capability specification

- [ ] 1.1 Write `specs/session-manageability/spec.md` defining policy input, Gridline-owned generation, per-setting locking, reporting of policy that is not in effect, screen lock policy as one policy, and non-interference with other sessions, and verify every requirement has at least one scenario and that no scenario names a file path or configuration format
- [ ] 1.2 Verify no requirement in `specs/session-manageability/spec.md` depends on a third-party configuration loader merging files or honoring directory order, since the baseline components do neither

## 2. Correct the existing precedence documentation

- [ ] 2.1 Correct `docs/configuration.md` so that it distinguishes XDG-aware applications, where `XDG_CONFIG_DIRS` precedence applies and a user override wins per setting, from the `hypr*` components, where first-match lookup means a user file replaces the administrator file wholesale, and verify the corrected document no longer states a single precedence rule for both cases
- [ ] 2.2 Record in `docs/configuration.md` that generated configuration is not a dpkg conffile, and verify the statement does not contradict the conffile rule that governs shipped editable defaults
- [ ] 2.3 Verify `docs/configuration.md` names the policy input as the supported way to set policy, and does not instruct an administrator to edit a component's own configuration file

## 3. Architecture

- [ ] 3.1 Add the policy layer to `docs/architecture.md`, placing it above the configuration layer, and verify the document states that generated configuration is an output of that layer rather than a hand-maintained input
- [ ] 3.2 Record in `docs/architecture.md` the mechanical reason the layer exists, naming `Hyprutils::Path::findConfig` first-match resolution and the absence of a lock mechanism, and verify the reason is stated as a verified property of the baseline rather than a design preference

## 4. Attribution and sequence

- [ ] 4.1 Assign every requirement in `specs/session-manageability/spec.md` to a downstream change in `proposal.md`, and verify the attribution table leaves no requirement in neither the assigned nor the deferred table
- [ ] 4.2 Record in `proposal.md` the preconditions this change imposes on `add-gridline-configuration-defaults`, `add-session-health-check` and `define-debian-package-architecture`, and verify each named change is one that exists in the documented sequence

## 5. Compliance traceability

- [ ] 5.1 Record in `docs/security-and-manageability.md` which CIS Ubuntu Level 1 Workstation screen-lock controls this capability makes satisfiable and which remain the platform's responsibility, and verify no control is claimed as satisfied by a requirement that only reports a condition rather than enforcing it
