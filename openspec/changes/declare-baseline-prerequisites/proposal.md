# Proposal

## Why

`gridline-session` ships `/usr/share/wayland-sessions/gridline.desktop`, a file only a display manager reads, and declares no display manager. Installed on a machine without one, the package's primary artifact is a file nothing opens, and nothing in the install says so.

The same question reaches further than one dependency. Gridline declares none of what the Ubuntu desktop baseline provides: no display manager, no GNOME session, no NetworkManager, no PipeWire. That is a deliberate position and it is recorded only as a table cell reading "Not applicable. GDM is the baseline display manager", which is an answer to a different question than the one an installer asks. The project's guarantee that the GNOME session remains a working fallback assumes a machine that has one, and nothing states that assumption where someone installing the package would meet it.

## What Changes

**This change ships implementation.**

- Declares what `gridline-session` needs to be usable at all, which at minimum is a display manager that offers Wayland sessions. Which package or virtual package expresses that is the first thing to settle: `x-display-manager` is provided by `gdm3`, `lightdm`, `sddm` and others, and it is the X one, so it does not say what Gridline means.
- States the baseline assumption where it belongs, so that a machine without an Ubuntu desktop is a machine the package declines or warns about rather than one it installs onto silently.
- Settles whether the GNOME fallback is a guarantee Gridline enforces or an assumption Gridline documents. These are different packages and different costs: enforcing it means depending on the desktop and accepting that installing Gridline installs GNOME, while documenting it means the fallback holds only on machines that already had one.
- Declares the relationship between Gridline's own packages, which is undeclared in one direction. `gridline-tools` depends on nothing, and `gridline doctor` reads `/usr/libexec/gridline/gridline-component-state`, which `gridline-session` ships. Removing `gridline-session` is an ordinary operation that leaves a `gridline` command which cannot answer. It declines clearly rather than crashing, which is why this has gone unnoticed, and a dependency that holds only by accident of what people happen to install is not declared.
- Settles whether the terminal is a dependency or a preference, because the project already treats it as both. `gridline-session` carries `Depends: kitty`, so removing the terminal removes the session; and the compositor configuration binds Return to `xdg-terminal-exec` rather than to a terminal, with a comment saying a user changes terminal by registering one. Both cannot be true. **Settled as a preference**: `Recommends: kitty`, so apt still installs it alongside Gridline and removing it no longer removes the session, with a second entry in the terminal list so that the preference degrades to the desktop's own terminal rather than to the arbitrary scan this project already eliminated once.

**Out of scope.** Anything about what Gridline does once a session starts. The session health check and the application launcher, which are their own changes.

**Out of scope, and the distinction matters.** Turning the three desktop shell components into recommendations. The terminal is separable because the session names a mechanism rather than a program and because an Ubuntu desktop always has another terminal; the bar, the notification daemon and the wallpaper daemon are each configured by a Gridline-owned file naming that specific component, and there is no equivalent already on the machine to fall back to. Treating them alike would trade a real guarantee for a symmetry.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

- `package-delivery`: two requirements are **added**. Nothing is modified: the capability already requires each package to declare what it needs, and already says a component whose absence blocks the session is a dependency rather than a recommendation. What it does not say is what a package should do about something it assumes and does not declare, nor what may be a recommendation rather than a dependency.

### Not modified, and the decision is recorded here

- `desktop-session` is **untouched**. The question of whether Gridline guarantees the GNOME fallback or assumes it was settled as assume-and-document, so the existing requirement keeps its meaning: installing or removing Gridline leaves the GNOME session as it was, which is non-interference rather than provision.

  Three things decided it. The archive's own convention: `sway`, `hyprland` and `labwc` each ship a session entry and none declares a display manager. The virtual package that would express it does not: `x-display-manager` is provided by `gdm3`, `xdm`, `wdm`, `slim`, `sddm` and `nodm`, and four of those six cannot offer a Wayland session at all. And the cost of the alternative: depending on `ubuntu-desktop` would install the whole GNOME desktop onto a machine whose owner had already chosen otherwise.

  What the session entry does without a display manager is nothing, which is the harmless failure. The gap this leaves is that installing onto a machine with no desktop says nothing, and the added requirement below is what closes it - in the package's own description rather than in its dependencies.

## Impact

- `packages/gridline/debian/control`: the declared dependencies of `gridline-session` and of `gridline-tools`, and the long description, which already carries the universe support gap and would carry this.
- `packages/gridline/src/config/gridline-xdg-terminals.list`: its single entry, and the comment explaining why there is only one. That comment reasons from the terminal being a dependency - "a second line would be unreachable" - so it becomes false the moment the dependency becomes a recommendation, and the two have to move together or the file misdescribes itself.
- `docs/component-ownership.md`: the GDM row, whose fallback cell reads "Not applicable" on the reasoning that without it there is no login screen for any session. That is true and it is not a reason to declare nothing.
- `docs/package-matrix.md`: the archive packages Gridline depends on, and the reasoning for each.
- `openspec/specs/package-delivery/spec.md`: whatever requirement settles this.
- Verification runs on a machine without a desktop as well as the LXD VM, because the condition this change is about cannot be observed on a machine that already has everything.

## Prior measurement

Recorded here because the change exists for a reason someone proposed and the measurement pointed elsewhere, and both halves are worth keeping.

The question arrived as "should Gridline depend on `ubuntu-desktop`", on the reasoning that the metapackage is present on an Ubuntu Desktop install and absent from the cloud image used for testing. Measured: the verification VM has `ubuntu-desktop`, `ubuntu-session`, `gnome-shell` and `gdm3` all installed, so the testing has not been running against a bare cloud image. And the spec requirement the dependency would serve reads as non-interference rather than provision: installing or removing Gridline leaves the GNOME session installed and functional.

What the question did expose is the display manager, which is narrower, cheaper, and actually missing.

**Removing Gridline does not touch the Ubuntu desktop, and that is measured rather than assumed.** Simulating `apt-get autoremove --purge` of all three Gridline packages removes 64, every one of them from the Hyprland and Qt6 stack. `ubuntu-desktop`, `ubuntu-desktop-minimal`, `gnome-shell`, `gdm3`, `mutter`, all three portals and `nautilus` stay. The four dependencies Gridline shares with the desktop are each held by something else: `xwayland` by `libmutter-18-0` and `ubuntu-session`, `xdg-terminal-exec` by `ptyxis` and `ubuntu-desktop-minimal`, and `libnotify-bin` and `ubuntu-wallpapers` by `ubuntu-desktop` directly. So declaring them costs nothing and risks nothing.

That first simulation was biased and the correction is worth keeping. Five packages appeared to survive removal for no reason - the terminal, the bar, the notification daemon, the wallpaper daemon and the application launcher - because all five were marked manual on the verification machine, having been installed by hand during development rather than pulled in as dependencies. Marked automatic, as a clean install would have them, the removal grows from 64 packages to 93 and three of the five go. The desktop is untouched in both readings, which is the answer to the question; but a measurement that reports the test machine's history as if it were the packaging's behaviour is one to notice before quoting it.

**The terminal has a fallback already, measured by accident.** A probe pointed Gridline's terminal list at an entry that does not exist, expecting to learn whether the session would then have no terminal at all. It opened `ptyxis`: `xdg-terminal-exec` does not fail on an unresolvable entry, it falls through. That is the fact that makes the terminal separable from the other components, and it was produced by a command that hung and had to be killed rather than by the measurement it was meant to be.

**What it falls through to was recorded wrong at first, and the correction is the reason for half of this change.** That reading was written down as "it opened ptyxis, which ubuntu-desktop-minimal brings", which reads as though the desktop's own terminal catches it. Nothing names ptyxis for a Gridline session: the desktop-keyed lists are keyed on `ubuntu`, `gnome` and `sway`, and Gridline's identity is `Gridline:Hyprland`. It was the directory scan, and the scan was right only because this machine has two terminals installed and one had just been removed. Counting them is what exposed it.
