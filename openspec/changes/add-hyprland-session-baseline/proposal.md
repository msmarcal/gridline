# Proposal

## Why

Selecting Gridline at the login screen today reaches the launcher, which reports a missing compositor adapter and hands the seat back. That is correct behavior and it was worth verifying, but it is the whole of what the session does. No Gridline user has seen a Gridline desktop.

`docs/compositor-adapter.md` already fixes what an adapter must report: nine required capabilities, two optional ones, and the rule that silence means absent. What it deliberately leaves open is the wire format, which it assigns to this change. The launcher's third branch says the same thing in code: it finds an executable adapter, cannot invoke it, and reports that as a launch failure.

So the gap is not a decision nobody has made. It is an interface nobody has written.

The baseline can support it. Every capability the contract requires is reachable through an interface Hyprland documents, measured against `hyprland` 0.53.3+ds-4 on the supported baseline: `ext_session_lock_v1` for `lock-signaling`, `ext_idle_notifier_v1` for `idle-reporting`, `zwlr_output_manager_v1` for `output-management`, `zwlr_screencopy_manager_v1` with `hyprland_toplevel_export_manager_v1` for `screencast`, and `hyprctl` for the rest. The two optional capabilities are reachable too. A contract that could not be satisfied would be a reason to revise the contract; this one can be.

## What Changes

**This change ships implementation.** It is the first since the package skeleton to deliver executable content and the first to write dependency fields into `debian/control`.

- Defines the adapter contract's wire format: how the launcher asks an adapter what it can do, how the adapter answers, and what the launcher does with each answer. The capability set and the reporting rules are already fixed and are not reopened.
- Adds the adapter layer at `packages/gridline/src/adapters/`, the single documented path `docs/compositor-adapter.md` names, installed under `/usr/libexec/gridline/adapters/`.
- Adds the Hyprland adapter as the first adapter, reporting each capability only where a documented Hyprland interface delivers it.
- Replaces the launcher's third branch. Instead of reporting that it cannot invoke the adapter, the launcher invokes it, compares the reported set against the required set, and either starts the compositor or reports the first missing capability by name.
- Adds the mechanical isolation check, which `docs/compositor-adapter.md` assigns here because this change creates both the boundary and the first thing inside it. The check decides adapter content by path alone.
- Declares the blocking dependencies this change makes real: `hyprland` and `xdg-desktop-portal-hyprland` on `gridline-session`. Both are in `universe`, and the support gap is recorded where the dependency is declared.
- Corrects the two stale statements in `debian/control` that describe `gridline-session` as shipping no content. The four describing `gridline-config` and `gridline-tools` stay, because those two still ship directories and no files.

**Out of scope.** No Hyprland configuration is shipped: keybindings, themes and defaults belong to `add-gridline-configuration-defaults`, and a session that starts with Hyprland's own defaults is the correct intermediate state. The portal configuration file stays with `add-desktop-portal-integration`. The locker and polkit agent selections stay with `add-session-lifecycle-integration`, so `lock-signaling` is reported as a capability without a lock component being chosen. No second adapter is written, and no CI job is added; `add-ci-vm-smoke-tests` owns re-running the isolation check.

## Capabilities

### New Capabilities

None. The adapter boundary is already owned by `compositor-abstraction` and the session's startup behavior by `desktop-session`.

### Modified Capabilities

- `compositor-abstraction`: gains requirements that the adapter answers a capability query in a defined form, and that the session refuses to start when a required capability is unreported. The existing requirements already say capabilities are discovered rather than assumed and that isolation is mechanically checkable; what is missing is the form of the answer and the consequence of a gap.
- `desktop-session`: gains a requirement that a session whose adapter reports every required capability reaches a usable desktop. The existing launch requirement covers the failure path, which is the half that could be verified before a compositor existed.

## Impact

- `packages/gridline/src/adapters/`: new, the adapter layer and the Hyprland adapter.
- `packages/gridline/src/session/gridline-session`: the third branch becomes an invocation rather than a report.
- `packages/gridline/debian/control`: `Depends` on `gridline-session`, and two corrected description statements.
- `packages/gridline/debian/rules` and `tools/`: the isolation check joins the ownership gate as a build-time gate.
- `docs/compositor-adapter.md`: the wire format section, which the document currently defers to this change by name.
- Verification runs on the LXD VM already carrying the supported baseline, where the session has so far only been observed failing.
