# Gridline: vision

## What Gridline is

Gridline is an optional Wayland tiling desktop session for Ubuntu, delivered as Debian packages and installed on top of a stock Ubuntu system.

It is not a distribution fork, not a replacement for Ubuntu Desktop, and not a configuration framework that rearranges a machine to suit itself. It is one more entry in the login screen's session list: select it to get a tiling desktop, select Ubuntu to get GNOME, and remove the packages to get the machine back exactly as it was.

## Supported baseline

| Property | Value |
|---|---|
| Distribution | Ubuntu 26.04 LTS |
| Architecture | x86_64 |
| Display manager | GDM |
| Initial compositor | Hyprland |

A single baseline is a deliberate choice. Supporting one release and one architecture well is worth more than partial support across several, and every requirement in this project is written to be verifiable on that baseline rather than in the abstract.

## The session model: additive and removable

Two properties define how Gridline relates to the system it is installed on.

**Additive.** Installing Gridline adds a session. It does not modify the GNOME session, change the default session for any user, alter shared system configuration, or take over components the rest of the desktop depends on. A user who installs Gridline and never selects it should not be able to tell it is there.

**Removable.** Removing Gridline returns the system to a stock Ubuntu desktop. No leftover configuration changes how GNOME behaves, and nothing outside administrator-modified files survives removal.

These are not aspirations. They are the reason several structural decisions look more conservative than they need to: Gridline defaults live in a Gridline-specific directory rather than the shared one, and the session's environment is assembled inside the session rather than exported system-wide.

## GNOME remains a supported fallback

**The regular Ubuntu GNOME session stays installed, selectable and supported at all times.** This holds while Gridline is installed, while it is running, and after it fails.

This matters most at the moment Gridline is least able to help. If a Gridline session cannot start, the user returns to GDM and selects Ubuntu GNOME, and that path works without any Gridline component running. Recovery never depends on the thing that just failed. See [recovery.md](recovery.md) for the full path.

The fallback is also why Gridline can afford to be ambitious elsewhere. A desktop that is the only way into a machine must be conservative; a desktop with a working escape hatch can take a risk, because the cost of a bad session is one selection at the login screen rather than a broken computer.

## Delivery

Gridline reaches users exclusively as Debian packages. Every capability a user gets comes from a package, never from an install script, a post-install download, or a manual file copy.

A future community Ubuntu Remix or ISO is a possible goal, and it would consume these same packages. An image may make Gridline easier to obtain; it must never be the only place a Gridline capability exists.

See [package-matrix.md](package-matrix.md) for the packages and what each owns.

## Non-goals

Project-wide, for the foreseeable future:

- **Omarchy visual parity.** Gridline is not a reimplementation of another project's look.
- **A custom shell.** No bespoke Quickshell-based shell is planned.
- **Per-device hardware automation.** Gridline does not detect and configure individual hardware models.
- **An application catalogue.** Gridline ships a session, not a curated set of applications.
- **AI features.**
- **Any change that replaces or removes GNOME.** This is the one non-goal that is a hard constraint rather than a priority call.

Two further exclusions are matters of sequence rather than principle: a status bar or launcher choice, and ISO or Remix work. Both may happen later; neither is in scope now.

## Where the constraints live

This document records intent. The binding form of each constraint is a requirement in `openspec/specs/`, written so a later change can be checked against it rather than against a reading of this page. Where the two ever disagree, the specs are authoritative.
