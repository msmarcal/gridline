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

So the sentence that would be natural here — that a user override wins over a shipped default — is true for the first column and misleading for the second. A user who writes any `~/.config/hypr/hyprlock.conf` does not override a Gridline default; they replace the only file that would have been read.

### Why policy is a separate layer

That property is why administrator policy cannot be expressed as a shipped default in a search path. An enforced setting needs a mechanism Gridline owns, which is specified by the `session-manageability` capability: policy input the administrator edits, a Gridline-owned generator that produces what the component actually reads, explicit per-setting locks, and a diagnostic that reports policy which is not in effect.

Two consequences for anyone working on configuration:

- **Generated configuration is not a conffile.** The conffile rule below governs shipped defaults an administrator is expected to edit. A file Gridline regenerates from policy input is an output; registering it as a conffile would make dpkg prompt about a file nobody is supposed to hand-edit.
- **The supported way to set policy is the policy input**, not editing the component's own configuration file. A file edited in place may be regenerated.

## Shipped defaults are conffiles

Defaults an administrator is expected to edit are registered as dpkg conffiles.

This is what implements "local changes survive upgrade" in `specs/package-delivery/spec.md`, including dpkg's prompt when a shipped default and a local edit both changed. Reimplementing preservation in a maintainer script would be more code and less predictable for anyone who already knows how Debian packages behave.

The practical consequence for a contributor: changing a shipped default is a packaging-visible act. An administrator who edited that file will be asked about it on upgrade, so a default is worth getting right rather than iterating on casually.

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
