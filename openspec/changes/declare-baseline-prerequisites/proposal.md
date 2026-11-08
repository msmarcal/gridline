# Proposal

## Why

`gridline-session` ships `/usr/share/wayland-sessions/gridline.desktop`, a file only a display manager reads, and declares no display manager. Installed on a machine without one, the package's primary artifact is a file nothing opens, and nothing in the install says so.

The same question reaches further than one dependency. Gridline declares none of what the Ubuntu desktop baseline provides: no display manager, no GNOME session, no NetworkManager, no PipeWire. That is a deliberate position and it is recorded only as a table cell reading "Not applicable. GDM is the baseline display manager", which is an answer to a different question than the one an installer asks. The project's guarantee that the GNOME session remains a working fallback assumes a machine that has one, and nothing states that assumption where someone installing the package would meet it.

## What Changes

**This change ships implementation.**

- Declares what `gridline-session` needs to be usable at all, which at minimum is a display manager that offers Wayland sessions. Which package or virtual package expresses that is the first thing to settle: `x-display-manager` is provided by `gdm3`, `lightdm`, `sddm` and others, and it is the X one, so it does not say what Gridline means.
- States the baseline assumption where it belongs, so that a machine without an Ubuntu desktop is a machine the package declines or warns about rather than one it installs onto silently.
- Settles whether the GNOME fallback is a guarantee Gridline enforces or an assumption Gridline documents. These are different packages and different costs: enforcing it means depending on the desktop and accepting that installing Gridline installs GNOME, while documenting it means the fallback holds only on machines that already had one.

**Out of scope.** Anything about what Gridline does once a session starts. The session health check and the application launcher, which are their own changes.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

- `package-delivery`: gains or modifies a requirement about what the session package declares. The capability already governs what each package owns and what no package does; what it does not say is what a package must declare to be installable onto a machine where it can work.
- `desktop-session`: possibly one requirement modified, depending on how the fallback question is settled. Its requirement that the GNOME session remains installed and supported is written as non-interference, "installing or removing Gridline SHALL leave" it so. If the project decides Gridline must guarantee rather than preserve the fallback, that requirement changes meaning and has to say so. If it decides to document the assumption instead, this capability is untouched and only `package-delivery` moves.

## Impact

- `packages/gridline/debian/control`: the declared dependencies of `gridline-session`, and its long description, which already carries the universe support gap and would carry this.
- `docs/component-ownership.md`: the GDM row, whose fallback cell reads "Not applicable" on the reasoning that without it there is no login screen for any session. That is true and it is not a reason to declare nothing.
- `docs/package-matrix.md`: the archive packages Gridline depends on, and the reasoning for each.
- `openspec/specs/package-delivery/spec.md`: whatever requirement settles this.
- Verification runs on a machine without a desktop as well as the LXD VM, because the condition this change is about cannot be observed on a machine that already has everything.

## Prior measurement

Recorded here because the change exists for a reason someone proposed and the measurement pointed elsewhere, and both halves are worth keeping.

The question arrived as "should Gridline depend on `ubuntu-desktop`", on the reasoning that the metapackage is present on an Ubuntu Desktop install and absent from the cloud image used for testing. Measured: the verification VM has `ubuntu-desktop`, `ubuntu-session`, `gnome-shell` and `gdm3` all installed, so the testing has not been running against a bare cloud image. And the spec requirement the dependency would serve reads as non-interference rather than provision: installing or removing Gridline leaves the GNOME session installed and functional.

What the question did expose is the display manager, which is narrower, cheaper, and actually missing.
