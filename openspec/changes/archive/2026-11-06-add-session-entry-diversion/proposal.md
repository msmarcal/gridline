# Proposal

## Why

Installing Gridline adds two entries to the login screen, and the user chose one of them. The compositor's own session entry is on the machine because `gridline-session` depends on the compositor: `apt-mark showmanual` does not list it. So a user who installed one desktop is offered two, and the one they did not ask for runs a session that inherits Gridline's keybindings and none of Gridline's components.

That second entry is also how a user reaches a session this project has an open finding about. Gridline's compositor defaults sit at the compositor's own system configuration path, which any session of that compositor reads, so selecting the entry produces a half-Gridline session: our bindings, no locker running, no polkit agent, no portal mapping, and a lock key bound to a signal nothing is listening for.

## What Changes

**This change ships implementation.**

- `gridline-session` diverts the compositor's session entries so the login screen offers Gridline and not also the bare compositor it is built on, using `dpkg-divert`, which is the mechanism Debian provides for one package displacing another's file.
- The diversion is registered when the package is installed and removed when it is, so a machine without Gridline offers the compositor's entry again.
- It covers every session entry the compositor ships, not only the visible one. A second entry is currently withheld by its own `TryExec` naming a binary the baseline does not carry, which is a fact about the baseline rather than a property of the entry.
- Records the cost. A package that diverts another maintainer's file by default is a serious objection in archive review, and `docs/security-and-manageability.md` already tracks official flavor candidacy as an ambition. This adds a second obstacle to it, knowingly.

**Out of scope.** Hiding any other session entry, and in particular the stock Ubuntu session, which is the fallback every recovery path in `docs/recovery.md` depends on. Nothing here makes a session unreachable to someone who wants it: the compositor remains installed and startable, and the administrator recipe already documented in `docs/recovery.md` works in the other direction for anyone who wants the entry back without removing Gridline.

## Capabilities

### Modified Capabilities

- `package-delivery`: one requirement is modified. **It currently forbids this change.** Its scenario "Influence on a third-party component uses a drop-in" requires that Gridline modify or replace no file another package owns, and a diversion relocates one. The requirement is right about the general case and was written before a case appeared that no drop-in can serve, because the only directory with precedence over the one holding session entries is `/usr/local/share`, which Debian Policy reserves for the local administrator. The modification admits one bounded exception with its own obligations rather than weakening the rule.
- `desktop-session`: gains one requirement, that the login screen offers Gridline and not the compositor Gridline is built on. The capability already requires that Gridline's own entry appear on install and disappear on removal, and says nothing about an entry Gridline's dependency brings with it.

### New Capabilities

None.

## Impact

- `packages/gridline/debian/gridline-session.postinst` and `.postrm`: new, the diversion and its removal. These are the first maintainer scripts in the project.
- `packages/gridline/tools/`: a check that every session entry the compositor ships is covered, since a diversion that names one of two stops working when the other becomes visible.
- `docs/package-matrix.md` and `docs/security-and-manageability.md` already record this decision and its cost; this change verifies what they describe rather than introducing it.
- `docs/recovery.md`: the administrator recipe it carries now works in the opposite direction, restoring an entry Gridline diverted rather than hiding one it does not.
- Verification runs on the LXD VM, where the login screen is the measurement and removal has to be exercised rather than reasoned about.

**One consequence beyond this change's own scope, named because it changes another change's work.** `add-desktop-shell-components` carries a scenario requiring that no session other than Gridline start its components. Diverting the entry is one of the two routes that satisfies it: with the entry gone, the accidental path into a half-Gridline session disappears. It does not close the underlying finding, which is that Gridline occupies the compositor's canonical configuration path, and that stays open and owned where it is recorded.
