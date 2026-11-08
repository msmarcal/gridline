# Design

## Context

See proposal.md for motivation. What the measurements settled, because three of the four decisions below were settled by them rather than by preference.

**The archive's convention is to declare nothing.** `sway`, `hyprland` and `labwc` each ship a session entry under `/usr/share/wayland-sessions/` and none of the three declares a display manager. They depend on libraries; `sway` recommends a polkit agent, a terminal and a menu, `hyprland` recommends XWayland and its portal backend.

**The virtual package that would express it does not.** `x-display-manager` is provided by `gdm3`, `xdm`, `wdm`, `slim`, `sddm` and `nodm`. Four of those six cannot offer a Wayland session at all, so depending on it would accept a machine where Gridline's session entry can never be selected. The proposal said this before it was measured; it is confirmed.

**Removing Gridline does not reach the Ubuntu desktop.** Simulated: `apt-get autoremove --purge` of all three Gridline packages removes 64 and leaves `ubuntu-desktop`, `gnome-shell`, `gdm3`, `mutter`, the portals and `nautilus` in place. The four dependencies shared with the desktop are each held by something the desktop itself declares.

**The terminal already has a fallback.** `xdg-terminal-exec` does not fail when the entry it is given cannot be resolved; it falls through, and on an Ubuntu desktop it finds `ptyxis`, which `ubuntu-desktop-minimal` brings. This was learned from a probe that hung and had to be killed after opening a terminal nobody asked for.

## Goals / Non-Goals

**Goals:**

- Every relationship a Gridline package depends on is declared, including between Gridline's own packages.
- What Gridline assumes and does not declare is stated where an administrator meets it.
- The terminal is replaceable without uninstalling the desktop that prefers it.

**Non-Goals:**

- Installing a desktop. Gridline does not put GNOME on a machine that chose not to have one.
- Making the three desktop shell components replaceable the same way. They are configured by name and have no mechanism to fall through to.
- Refusing to install. A package that declines on a machine where its artifact is inert would be refusing on a condition that can change five minutes later.

## Decisions

### The baseline is assumed and said, not declared

`gridline-session` declares no display manager and no desktop. Its long description says what it assumes: that the session entry is read by a display manager, that Gridline installs none, and that without one the entry is inert rather than broken.

The alternative, `Recommends: gdm3 | sddm`, was considered and rejected on the measurement above. It departs from what every comparable package in the archive does, and the first alternative in that list is the one apt installs, so a machine that had chosen `sddm` and lost it would get `gdm3` back from a package that has no business choosing.

**Inert is the right failure and worth naming.** A session entry nothing reads costs nothing and breaks nothing. The only harm is silence, and silence is what the description fixes.

### The terminal becomes a recommendation, and the list grows a second line

`Recommends: kitty`, so apt still installs it with Gridline and removing it no longer removes the session.

This is allowed by the requirement that already exists - a component whose absence blocks the session must be a dependency, and the terminal's absence does not block it, measured - but allowed is not the same as right, which is why this change adds the requirement that says *why* it should be a recommendation.

**The second line in the terminal list is not optional and the two cannot ship apart.** `gridline-xdg-terminals.list` holds one entry, and its comment justifies that by the terminal being a dependency: *"a second line would be unreachable."* The moment it stops being a dependency, that sentence is false and the file holds a preference that can fail to resolve. So the list gains `org.gnome.Ptyxis.desktop:new-window`, which is what Ubuntu lists for itself, and the comment is rewritten to say what it now does.

Without that second line the failure is not "no terminal" - it is worse and quieter. `xdg-terminal-exec` falls through to an unordered scan of every installed application, and this project measured that scan once already: 114 candidates with three terminals at positions 9, 76 and 102, so installing an unrelated application could change which terminal a key opens.

### `gridline-tools` declares `gridline-session`

The `gridline` command reads `/usr/libexec/gridline/gridline-component-state`, which `gridline-session` ships, and `gridline-tools` declares nothing. Removing the session is an ordinary operation that leaves a `gridline` command which cannot answer.

It declines with a clear message rather than crashing, which is why nobody noticed. That is good behaviour and not a substitute for the declaration: the requirement already in `package-delivery` says each package declares what it requires, so this is a defect against a requirement that predates the change rather than something this change invents.

**Which direction, and it is only one.** `gridline-tools` depends on `gridline-session`; the reverse is not true and must not be added, because a session has no need of the diagnostic and adding it would make the command unremovable.

### The descriptions are part of the deliverable, not documentation about it

Two of the three package descriptions said "no content yet" while shipping nine files and a command, and the page that checks descriptions against packages asserted they were still accurate. Those were corrected ahead of this change, in the commit that found them.

What this change adds is the requirement that makes the correction stick: a description names what is there rather than what is intended, so the comparison a reader makes against `dpkg-deb -f` is one that can fail. A rule nobody can fail is not a rule.

## Risks / Trade-offs

- **Declaring nothing means an install onto a server says nothing useful at install time.** → The description is read by `apt show` and printed by `apt` before confirming an install, so it is not hidden; and a package that refused would be refusing on a condition a later `apt install gdm3` changes.
- **`Recommends` is skipped by `--no-install-recommends`, so that install gets no terminal of Gridline's choosing.** → It gets the machine's, through the mechanism, which is the behaviour this change is making explicit rather than a regression. On a machine with no terminal at all the binding fails visibly, which is the same as today on a machine where the dependency was force-removed.
- **Two places now describe what Gridline needs: the dependencies and the description.** → The added requirement says what happens when one becomes the other, because this project has already shipped a page asserting a fact about a package that had stopped being true.

## Open Questions

- Whether the three desktop shell components should eventually reach their roles through a mechanism too, which would make them replaceable on the same terms. There is no freedesktop mechanism for "the bar this system uses", so this is a question about whether Gridline invents one, and it is not this change's.
