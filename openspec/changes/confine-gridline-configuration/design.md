# Design

## Context

See proposal.md for motivation. Four facts, three of them measured while implementing other changes.

**The violation.** `gridline-config` declares `/etc/xdg/hypr/hyprland.conf` and owns `/etc/xdg/hypr/`, confirmed by `dpkg -S`. `package-delivery` requires that a Gridline package declare ownership only of paths whose name identifies them as Gridline's. Neither segment does.

**Why nothing caught it.** `tools/check-path-ownership` compares the staged trees of the Gridline packages against each other, looking for a path two of them claim; a path only one claims passes however it is named. `tools/check-adapter-isolation` scans `src/` and `tools/`, and the manifest line lives in `debian/`, which it excludes. The requirement's own scenario about catching a declared path at build time is written for a path an archive package owns, and this one is owned by nobody else.

**The compositor accepts an explicit path.** `--config FILE`, `-c FILE`, read from its own usage output. This is the same mechanism `add-gridline-configuration-defaults` used to answer a different question and did not apply here.

**The loader exists only to be found.** `/etc/xdg/hypr/hyprland.conf` contains one `source` line. It is not configuration; it is a way of getting configuration read, chosen when naming the path was not considered.

## Goals / Non-Goals

**Goals:**

- Every path a Gridline package declares carries a Gridline-identifying name, and a gate says so.
- Gridline's compositor defaults apply to a Gridline session and to no other session of that compositor.
- The customization story survives in a form that is at least as good as the one it replaces.

**Non-Goals:**

- The bar, the notification daemon and the wallpaper daemon. `add-desktop-shell-components` owns them and waits on the pattern settled here.
- Changing `package-delivery`. The requirement was right; what was missing was enforcement.
- The other two findings in `docs/compositor-adapter.md`. Moving a configuration file touches neither.

## Decisions

### Name the path instead of occupying the component's

The adapter's `start` verb passes the configuration path to the compositor. The defaults live under `/etc/xdg/gridline/`, which is already Gridline's and already carries them; what goes away is the second copy of the arrangement at `/etc/xdg/hypr/hyprland.conf`.

Three problems close together, which is the reason to do this as one change rather than three:

- The declared path becomes Gridline-named, which is the requirement.
- The defaults stop being visible to any other session of that compositor, which is the open finding in `docs/compositor-adapter.md`. A session started outside Gridline searches and finds nothing of ours.
- `/etc/xdg/hypr/` stops being a directory Gridline creates and owns, so an administrator who wants a system-wide compositor configuration independent of Gridline can have one.

### What this does to the user's own configuration, and why it is not worse

Under the old arrangement the compositor searched, found Gridline's loader, and a user who wrote their own file replaced it wholesale. The customization story was that they copy the loader and `source` Gridline's defaults.

Under the new one the compositor is told a path, so it does not search at all, and a user's own file would never be read. That is worse, not better, and it has to be answered rather than accepted: naming a path cannot mean the user loses the ability to configure their own compositor.

The adapter therefore names Gridline's path **only when the user has none of their own**, and the search paths it checks are the component's own, not a guess. A user with a configuration gets exactly what they would get without Gridline installed; a user with none gets Gridline's defaults. The scenario that says so is in the delta.

This is Gridline-authored glue, like the lock entry point, and it is deliberate: the component has no mechanism that expresses "use mine unless theirs exists", so the session supplies it. The cost is that the precedence rule becomes Gridline's behaviour rather than the component's, which is recorded rather than hidden.

### The terminal is chosen by a file keyed on Gridline's desktop name

The same principle reaches a second component, which is why it is here rather than in a change of its own.

Two things were measured and both are wrong today. `docs/package-matrix.md` says the terminal is declared because the baseline guarantees none; the baseline **recommends** one, `ptyxis`, in both desktop metapackages, and apt installs recommends by default. The original measurement grepped the recommends list for the words `console` and `terminal`, and the package is named neither. And with two terminals present the launcher selects Gridline's, so installing Gridline changes which terminal the machine opens, by winning an ordering rule nobody chose.

The launcher reads a preference file keyed on the desktop name, read from its source: `${XDG_DATA_DIRS}/xdg-terminal-exec/<desktop>-xdg-terminals.list`, with `<desktop>` iterating the lowercased elements of `XDG_CURRENT_DESKTOP`. Gridline's identity makes that `gridline-xdg-terminals.list`.

That file is the answer to both problems at once. It names the terminal rather than hoping an ordering picks it, and because it is keyed on the desktop name it applies in a Gridline session and nowhere else, so another session keeps whatever it had. It is a Gridline-named file in a directory the component reads, which is the drop-in shape `package-delivery` blesses and the same shape as `gridline-portals.conf`.

**Declaring a terminal is still right, for a different reason than the one recorded.** Recommends is not a guarantee: an install made with `--no-install-recommends` has none, and the scenario that the keyboard works on first login would fail on it. What changes is the justification, which this change corrects rather than leaves to read as measured when it was guessed.

**The preference lists one entry and no fallback.** The declared terminal is a dependency, so a machine without it is a machine without `gridline-session`, and a second entry would be unreachable. Shipping an unreachable line to feel safe is the kind of thing this project has been removing.

### A gate for Gridline-named declared paths

The requirement has been in force since `establish-gridline-vision` and nothing enforced it. A gate reading the install manifests and failing on a declared path whose name identifies no Gridline ownership would have caught this on the day it shipped.

What counts as Gridline-named needs stating rather than assuming, because the obvious rule is wrong in both directions. `/etc/xdg/gridline/...` qualifies. `/usr/share/xdg-desktop-portal/gridline-portals.conf` qualifies on the filename while its directory belongs to another component, which is the drop-in shape the requirement explicitly blesses. `/usr/share/wayland-sessions/gridline.desktop` is the same shape. So the rule is that **some segment of the declared path carries the project's name**, not that the whole path does.

The gate is the part of this change most likely to be written loosely, since every path in the tree today passes it. Planting a failing path is how it is known to work.

## Risks / Trade-offs

- **The precedence rule moves from the component to Gridline.** → Recorded in the design and in the delta, and the behaviour a user sees is the one they would expect: their file wins. The alternative is naming a path unconditionally, which takes their compositor configuration away from them.
- **A change that edits what another change shipped and archived.** → The archived change is not wrong about what it verified; it verified a mechanism against the scenarios it had. This one found a requirement it did not check against, and the fix belongs where the fix is rather than in a retrospective.
- **Removing a conffile an administrator may have edited.** → `/etc/xdg/hypr/hyprland.conf` is a conffile. An upgrade that stops shipping it leaves dpkg handling the edited case, and the change has to say what happens to an administrator who edited it rather than discovering it at verification.

## Open Questions

- Whether the compositor's search paths can be checked cheaply enough for the adapter to decide, and what it does when it cannot tell. Measurable before implementation.
- What dpkg does with a conffile the package stops shipping, in the unmodified and the modified case. These differ, and the second is the one that matters to someone who edited it.
