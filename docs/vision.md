# Gridline: vision

## What Gridline is

Gridline is an optional Wayland tiling desktop session for Ubuntu, delivered as Debian packages and installed on top of a stock Ubuntu system.

It is not a distribution fork, not a replacement for Ubuntu Desktop, and not a configuration framework that rearranges a machine to suit itself. It is one more entry in the login screen's session list: select it to get a tiling desktop, select Ubuntu to get GNOME, and remove the packages to get the machine back exactly as it was.

**What distinguishes Gridline is not tiling, and not polish.** Tiling sessions for Linux already exist, and projects built on rolling bases ship them with a faster cadence and a stronger aesthetic than Gridline will have.

Gridline is a third option for a specific person: someone who already uses Ubuntu and likes using it, works in the terminal, saw Omarchy and wanted that. Their choices today are to leave Ubuntu for a rolling base, or to assemble the stack by hand and maintain it alone indefinitely. Gridline is that experience delivered as Debian packages on the Ubuntu they already run, with the Ubuntu session intact beside it.

The audience is small, the way Omarchy's is. That is the nature of the project rather than a problem for it to solve, and it is why the project optimizes for being trustworthy on one working machine rather than for reach.

Everything demanding in this project follows from that, and not from a compliance requirement. Packages instead of install scripts, GNOME always present, clean removal and a single compositor boundary exist because this runs on the machine its user works on, where a failed experiment must cost one session selection and not a working day.

The gaps between what Gridline can promise and what the archive provides are recorded in [security-and-manageability.md](security-and-manageability.md). The largest is that the compositor stack lives in `universe` and carries no Canonical security-update commitment. Gridline documents that gap rather than hiding it.

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

- **Omarchy visual parity.** Gridline is not a reimplementation of another project's look. It adopts the usability of the Omarchy 3.x experience, built from archive packages; see [user-experience.md](user-experience.md) for the inventory and the exclusions.
- **A custom shell.** No bespoke Quickshell-based shell is planned. Omarchy 4.0 moved to one, and it is not in the Ubuntu archive; chasing it would mean maintaining a desktop shell outside the archive.
- **Per-device hardware automation.** Gridline does not detect and configure individual hardware models.
- **An application catalogue.** Gridline ships a session, not a curated set of applications.
- **AI features.**
- **Any change that replaces or removes GNOME.** This is the one non-goal that is a hard constraint rather than a priority call.

Two further exclusions are matters of sequence rather than principle: a status bar or launcher choice, and ISO or Remix work. Both may happen later; neither is in scope now.

## The name

The project is **Project Gridline**, and its address is `https://gridline-desktop.org`. Packaging metadata and documentation use that host; no other domain is the project's address.

"Gridline" is a common word in software, and the name was checked rather than assumed. The findings are recorded here so that the decision does not have to be re-argued from scratch.

**What exists.** `GRIDLINE` is a registered US trademark (reg. 5057814, registered 2016-10-11) held by Redwing Technology Corporation, covering international classes 035, 042 and 045. Class 042 includes "design and development of computer hardware and software". Two further applications are pending under the same word mark. At least six unrelated companies and projects use the name in software: fleet telematics, private-market investment infrastructure, document billing, an AEC intelligence layer, an open-source database GUI, and a spreadsheet application.

**Why the name is kept anyway.** Three reasons, in order of weight:

1. The package names are available. `gridline`, `gridline-desktop` and `gridline-session` are unused as source package names in both the Ubuntu archive and Debian, so there is no technical packaging conflict to resolve.
2. None of the existing users is a Linux desktop environment. Market confusion between a Wayland tiling session and fleet telematics or an investment platform is implausible, which is the question a trademark dispute actually turns on.
3. A non-commercial open-source project is a weak target for an infringement claim, though not an immune one.

**What this costs.** The risk is not zero: a registered mark in class 042 covers software, so a claim is possible if the project becomes visible under the bare word. Two precautions follow from that, and both are cheap:

- Public communication uses **"Project Gridline"** or **"Gridline Desktop"**, never "Gridline" alone. A compound name sits further from a registered word mark than the word by itself.
- No attempt is made to register "Gridline" as a trademark for this project. With one registration and two pending applications on the same word, the likely outcome is refusal.

**The compound form governs prose, not identifiers.** It applies to the project name, documentation, the website and public communication. Technical identifiers keep the bare word and do not change: the `gridline` command users type, the `gridline` source package, the `gridline-` binary package prefix the tooling globs on, and the domain. Renaming those would be a misreading of this precaution. That `gridline-desktop` is both the public compound form and the reserved metapackage name is a convenience, not a conflict.

This section is a project record rather than the output of a task in this change. No task or requirement covers the name search, and none should be invented to cover it retroactively; it is written here because the next change would otherwise re-derive it.

`gridline` is already taken on PyPI, npm and crates.io; `gridline-desktop` is free on all three. If the project ever publishes to a language registry, it uses the compound form there too.

## Where the constraints live

This document records intent. The binding form of each constraint is a requirement in `openspec/specs/`, written so a later change can be checked against it rather than against a reading of this page. Where the two ever disagree, the specs are authoritative.
