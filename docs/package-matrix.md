# Package matrix

The binary packages Gridline is delivered as, what each owns, and which capability requires it.

The authoritative form of these boundaries is `openspec/specs/package-delivery/spec.md`. This page is the readable summary; where the two disagree, the spec wins.

## Packages

| Package | Owns | Required by capability | Status |
|---|---|---|---|
| `gridline-session` | The GDM session entry, the session launcher, the core integration glue, and the compositor adapter layer | `desktop-session` (the session exists and starts), `platform-integration` (the session reaches the desktop stack), `compositor-abstraction` (the adapter lives here) | Planned |
| `gridline-config` | Shipped defaults under `/etc/xdg`, default keybindings, default themes, and the portal interface-to-backend mapping | `desktop-session` (session-scoped configuration), `platform-integration` (deterministic portal backend selection) | Planned |
| `gridline-tools` | The `gridline` command, `gridline doctor`, and the theme and diagnostic commands | `platform-integration` (capability gaps are diagnosable), `compositor-abstraction` (the command surface stays compositor-neutral) | Planned |
| `gridline-desktop` | Nothing. A metapackage that pulls in the packages making up a complete Gridline desktop | `package-delivery` (the name is reserved) | **Reserved, not delivered** |

Every binary package uses the `gridline-` prefix. See [packages/gridline/README.md](../packages/gridline/README.md) for why the prefix is a convention the build tooling relies on rather than a naming preference.

## Boundaries that are easy to get wrong

Three boundaries sit close enough together to be worth stating explicitly, because `specs/package-delivery/spec.md` requires that no file be claimed by two packages.

**Themes are split by kind, not by subject.** Theme *data* is shipped defaults, so it belongs to `gridline-config`. The theme *commands* that apply or list those themes are part of the command surface, so they belong to `gridline-tools`. A theme feature therefore touches two packages, and neither owns the other's files.

**The adapter is integration glue, not a tool.** The compositor adapter ships in `gridline-session` because that package owns integration. Putting it in `gridline-tools` would force the diagnostic commands to carry compositor knowledge, which `specs/compositor-abstraction/spec.md` forbids.

**The portal mapping is configuration, not session logic.** The interface-to-backend mapping is a shipped default, so `gridline-config` owns it, even though the session is what consumes it.

## What no package does

These apply to every package above, and come straight from `specs/package-delivery/spec.md`:

- **Nothing is fetched at install time.** No maintainer script downloads code or configuration to finish its own installation.
- **No file is claimed twice.** The packages install together without file conflicts, and this is checked at build time rather than discovered at install time.
- **Local changes survive upgrade.** Defaults an administrator is expected to edit are dpkg conffiles.
- **Removal returns the system to stock.** After removing every Gridline package, no session entry remains, GNOME works, and nothing Gridline owned is left behind apart from administrator-modified configuration.

## On `gridline-desktop`

The name is reserved by the `establish-gridline-vision` change and **no `gridline-desktop` package is produced by it**. The metapackage is delivered later, when the component set it would depend on is stable, which no currently scheduled change reaches. It is listed in the deferred requirements table in that change's `proposal.md` for exactly this reason.

When it does ship, it carries no executable, configuration or data files of its own beyond packaging metadata. A metapackage that acquires content has stopped being a metapackage.

## Source packages

All four binary packages are built from one source package at `packages/gridline/`. That decision, and the room the layout leaves for additional source packages, is recorded in [packages/gridline/README.md](../packages/gridline/README.md).
