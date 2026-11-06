# Configuration

How Gridline's configuration is resolved, and why its defaults are kept out of the shared system search path.

The binding requirements are in `openspec/specs/desktop-session/spec.md` and `openspec/specs/package-delivery/spec.md`.

## The rule that shapes everything else

**Gridline defaults never alter configuration resolution for a GNOME session.** A user who installs Gridline and logs into GNOME gets exactly the configuration resolution they had before, and removing Gridline changes nothing about it either.

This is not a nicety. `/etc/xdg` is a *shared* search path: every XDG-aware application in every session reads it. A Gridline theme or GTK default dropped at the top level of `/etc/xdg` would be read by the GNOME session too, so installing Gridline could change the appearance or behavior of the user's GNOME desktop, and removing it could change it back. That would break the additive and removable guarantees in `specs/desktop-session/spec.md`, which are the whole basis of GNOME remaining a supported fallback.

## How it works

`gridline-config` installs its defaults into a **Gridline-specific directory beneath `/etc/xdg`**, not at its top level.

The session launcher **prepends that directory to `XDG_CONFIG_DIRS` for the Gridline session only**. The variable is assembled inside the session's own environment; nothing is exported system-wide, written to a shared profile script, or set for the display manager.

The consequence is the one that matters: inside a Gridline session, Gridline's defaults are visible to applications. Inside a GNOME session, that directory is not on the search path, so it does not exist as far as configuration resolution is concerned.

## Precedence

There are two resolution models in a Gridline session, not one. Which applies depends on how the reading component looks up its configuration, and conflating them is the mistake this section exists to prevent.

### XDG-aware applications

For applications that read `XDG_CONFIG_DIRS` and merge per setting, precedence runs from highest to lowest:

| Precedence | Source | Owner |
|---|---|---|
| 1 | `$XDG_CONFIG_HOME`, or `~/.config` when unset | The user |
| 2 | Gridline's directory beneath `/etc/xdg` | `gridline-config`, editable by the administrator |
| 3 | The shared `/etc/xdg` | Ubuntu and other packages |
| 4 | Application built-in defaults | Each application |

A user override wins over a shipped Gridline default for that setting, which is ordinary XDG behavior and not something Gridline adds. An administrator editing a shipped default affects every user of the Gridline session on that machine, and nothing in any other session.

In a GNOME session, level 2 is absent and the remaining levels are unchanged.

### The `hypr*` components

The compositor, locker and idle manager do **not** follow that model. They resolve configuration through `Hyprutils::Path::findConfig`, which checks `$XDG_CONFIG_HOME`, then `$HOME/.config`, then each entry of `XDG_CONFIG_DIRS`, then `/etc/xdg`, and **returns the first file it finds and stops**.

The difference is not a detail of ordering. It changes what a user file does:

| | XDG-aware application | `hypr*` component |
|---|---|---|
| User sets one key | That key overrides; other administrator values still apply | The administrator file is **never read**; every value in it is lost |
| Administrator can enforce a value | Only if the user has not set it | Not by file placement at all |
| Unapplied policy is visible | Values are traceable per source | Indistinguishable from policy being absent |

So the sentence that would be natural here - that a user override wins over a shipped default - is true for the first column and misleading for the second. A user who writes any `~/.config/hypr/hyprland.conf` does not override a Gridline default; they replace the only file that would have been read. The example is the compositor's own configuration because that is where the model applies: the screen locker Gridline selected does not use it.

### What Gridline does about first file wins

The property above would make a shipped default all or nothing: read when the user has written no configuration, and discarded entirely the moment they write one. So Gridline does not rely on being found at all. The session names its configuration path when it starts the compositor, and names it **only when the compositor's own search would find nothing**:

| Path | What it is | Owner |
|---|---|---|
| `/etc/xdg/gridline/compositor/defaults.conf` | The defaults, named by the session at startup | `gridline-config`, a conffile |

Three consequences, and the second is the one that makes this different from placing a file where the component looks:

- **A user's own configuration wins by existing.** If the compositor would find anything of its own, on any path it searches, the session names nothing and the compositor resolves exactly as it would on a machine without Gridline. The user does not have to opt out of Gridline's defaults; they opt in by having none.
- **Gridline's configuration reaches a Gridline session and no other.** A session of the same compositor started outside Gridline finds nothing of ours. Measured: the configuration such a session produces is byte for byte the file this machine generated before Gridline was installed.
- **No Gridline file sits in the compositor's directory.** `/etc/xdg/hypr/` is free for an administrator who wants a system-wide compositor configuration that is not Gridline's, and `package-delivery`'s rule that every declared path is Gridline-named is satisfied rather than broken.

**The precedence rule here is Gridline's, not the component's**, and that is worth saying plainly. The compositor has no way to express "use mine unless theirs exists", so the session supplies it. What a user sees is what they would expect, their file winning, but the mechanism behind it is ours.

**This is not enforcement and is not meant to be.** A user who writes their own configuration gets it, whole, with none of Gridline's. A shipped default is something a user may decline, and here declining is simply having a configuration.

**Which configuration is in effect is answerable.** Nothing in the compositor reports the file it loaded, so Gridline's defaults set `GRIDLINE_CONFIG` through the compositor's `env` keyword, and an `exec-once` pushes it into the systemd user manager so it reaches a terminal the user opens. A session running without Gridline's defaults has neither, which is what makes the absence detectable rather than discovered by a key that does nothing.

**An earlier arrangement placed a loader at the compositor's own search path**, with the defaults beside it and a `source` line joining them. It worked, and it had two faults that only appeared later: the loader was a path Gridline declared and could not name as its own, which `package-delivery` forbids, and every session of that compositor read it, not only Gridline's.

### Why policy is a separate layer

That property is why administrator policy cannot be expressed as a shipped default in a search path. An enforced setting needs a mechanism Gridline owns, which is specified by the `session-manageability` capability: policy input the administrator edits, a Gridline-owned generator that produces what the component actually reads, explicit per-setting locks, and a diagnostic that reports policy which is not in effect.

Two consequences for anyone working on configuration:

- **Generated configuration is not a conffile.** The conffile rule below governs shipped defaults an administrator is expected to edit. A file Gridline regenerates from policy input is an output; registering it as a conffile would make dpkg prompt about a file nobody is supposed to hand-edit.
- **The supported way to set policy is the policy input**, not editing the component's own configuration file. A file edited in place may be regenerated.

## Shipped defaults are conffiles

Defaults an administrator is expected to edit are registered as dpkg conffiles.

**The test is authorship, not location.** A file an administrator writes or is expected to edit is a conffile: shipped defaults, policy input, and lock declarations. A file Gridline produces from those is not, and it is removed when its owning package is removed. This is the same rule stated from the packaging side in [packaging.md](packaging.md), which carries the full table and the reason for each row. The two documents state one rule, not two similar ones.

This is what implements "local changes survive upgrade" in `specs/package-delivery/spec.md`, including dpkg's prompt when a shipped default and a local edit both changed. Reimplementing preservation in a maintainer script would be more code and less predictable for anyone who already knows how Debian packages behave.

The practical consequence for a contributor: changing a shipped default is a packaging-visible act. An administrator who edited that file will be asked about it on upgrade, so a default is worth getting right rather than iterating on casually.

### When Gridline stops shipping a default

A default that goes away is a case the rule above does not cover, and dpkg's own
behaviour is not what a reader would guess. Dropping the file from the package
does **not** remove it from a machine that already has it: an obsolete conffile
is left in place, whether or not anybody edited it, so the component that reads
it keeps reading it. Measured on the baseline rather than taken from policy, in
both the edited and the unedited case.

So Gridline declares the removal with `dpkg-maintscript-helper rm_conffile`, and
the two cases then differ in the way an administrator would want:

| On upgrade | What happens |
|---|---|
| You never edited the file | `Removing obsolete conffile <path>`, and it is gone |
| You edited it | `Obsolete conffile <path> has been modified by you`, and your version is kept as `<path>.dpkg-bak` |

**Nothing you wrote is destroyed, and nothing you wrote is still in effect.** The
backup is not read by the component, because it no longer carries the filename
the component searches for. If you want your changes back, move them into your
own configuration, or into a file at the original path, which is then yours
rather than a Gridline default that happens to be editable.

This happened once, to `/etc/xdg/hypr/hyprland.conf`. Gridline used to place the
compositor's defaults where the compositor searches, and stopped, so that its
configuration reaches a Gridline session and no other session of that
compositor. [recovery.md](recovery.md) records what that changed for anyone
running the compositor outside Gridline.

### The desktop shell components

The bar, the notification daemon and the wallpaper daemon follow the same rule
as the compositor and reach it by a different route, because what starts them
is a systemd unit the archive ships rather than a launcher Gridline wrote.

Their shipped defaults live under `/etc/xdg/gridline/shell/`. A drop-in per
unit routes `ExecStart` through `/usr/libexec/gridline/gridline-shell`, and
that program decides, at start time, against two questions:

| Question | If no |
|---|---|
| Is this a Gridline session, read from `XDG_CURRENT_DESKTOP`? | exec the component with no arguments |
| Would the component find configuration of its own? | name Gridline's path |

**The path is not in the drop-in on purpose.** A drop-in rewrites `ExecStart`
for every session on the machine, so naming the configuration there would make
Gridline's defaults reach these components started outside Gridline. That is
the same leak the compositor's loader had, in a second place, and the session
check is what keeps the drop-in from meaning anything elsewhere.

**Which search bases count is narrower than each component's own search**, and
the rule is measured per package: a system base counts only where the
component's package ships nothing into it, so that a file being there means
somebody put it there. The bar is the one of the three that ships its own
system configuration, at `/etc/xdg/waybar/`, so that base is not consulted.
Treating it as a user's choice made Gridline's bar configuration unreachable on
every machine, because the archive's default is always present.

What that gives up is worth stating: an administrator cannot override
Gridline's bar configuration by editing the archive's file, because nothing can
tell that edit apart from the file dpkg installed. The supported route is
editing Gridline's own conffile, which is what the conffile rule below makes it
for.

**The wallpaper daemon's format is not the one its documentation shows.** The
version the archive ships rewrote it: `preload` is gone, and `wallpaper` is a
category with a block rather than a one-line value. A file in the older syntax
parses without complaint and displays nothing, indistinguishable at runtime
from no configuration at all, so a change to that file is checked against a
running session rather than against an exit code.

## What belongs here and what does not

`gridline-config` owns the shipped defaults, the default keybindings, the default themes, and the portal interface-to-backend mapping.

It does not own:

- The commands that apply or list themes. Those are the command surface, owned by `gridline-tools`. Theme data and theme commands live in different packages on purpose; see [package-matrix.md](package-matrix.md).
- Anything compositor-specific. Keybindings are expressed as Gridline operations, which the active adapter carries out. This is required by `specs/compositor-abstraction/spec.md`, and it is why switching compositors does not rewrite the keybinding defaults.

The *contents* of the defaults, keybindings and themes are not settled by this change. They are owned by `add-gridline-configuration-defaults`, along with the user-facing configuration surface. What is settled here is where they live and how they resolve.

## Portal backend selection

The portal interface-to-backend mapping is shipped configuration rather than session logic, because it is a default an administrator may need to inspect or change.

It exists because `specs/platform-integration/spec.md` requires a deterministic choice when more than one installed backend implements the same portal interface. Shipping an explicit mapping makes that choice reviewable and diagnosable. Relying on backend start order makes it neither, and produces a desktop whose screen sharing works or does not depending on which service won a race.

The compositor-dependent interfaces go to the compositor's own backend; the rest go to the GTK backend. Which interfaces fall in which group is owned by `add-desktop-portal-integration`.
