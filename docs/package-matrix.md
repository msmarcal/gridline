# Package matrix

The binary packages Gridline is delivered as, what each owns, and which capability requires it.

The authoritative form of these boundaries is `openspec/specs/package-delivery/spec.md`. This page is the readable summary; where the two disagree, the spec wins.

## Packages

| Package | Owns | Required by capability | Status |
|---|---|---|---|
| `gridline-session` | The GDM session entry at `/usr/share/wayland-sessions/gridline.desktop`, the session launcher at `/usr/libexec/gridline/gridline-session`, the core integration glue, and the compositor adapter layer | `desktop-session` (the session exists and starts), `platform-integration` (the session reaches the desktop stack), `compositor-abstraction` (the adapter lives here) | **Entry and launcher shipped**; glue and adapter planned |
| `gridline-config` | Shipped defaults under `/etc/xdg`, default keybindings, default themes, the portal interface-to-backend mapping, `gridline-portals.conf`, the administrator's policy input, the lock declarations, and the destination of the generated configuration | `desktop-session` (session-scoped configuration), `platform-integration` (deterministic portal backend selection), `session-manageability` (policy input and locks) | Planned |
| `gridline-tools` | The `gridline` command, `gridline doctor`, the theme and diagnostic commands, and the generator that produces the effective configuration | `platform-integration` (capability gaps are diagnosable), `compositor-abstraction` (the command surface stays compositor-neutral), `session-manageability` (the generator) | Planned |
| `gridline-desktop` | Nothing. A metapackage that pulls in the packages making up a complete Gridline desktop | `package-delivery` (the name is reserved) | **Reserved, not delivered** |

Every binary package uses the `gridline-` prefix. See [packages/gridline/README.md](../packages/gridline/README.md) for why the prefix is a convention the build tooling relies on rather than a naming preference.

## Boundaries that are easy to get wrong

Three boundaries sit close enough together to be worth stating explicitly, because `specs/package-delivery/spec.md` requires that no file be claimed by two packages.

**Themes are split by kind, not by subject.** Theme *data* is shipped defaults, so it belongs to `gridline-config`. The theme *commands* that apply or list those themes are part of the command surface, so they belong to `gridline-tools`. A theme feature therefore touches two packages, and neither owns the other's files.

**The generated configuration is split the same way.** `gridline-config` owns the destination, and `gridline-tools` owns the generator that writes into it, because the destination is configuration and the generator is a command. `gridline-config` installed on its own owns an empty destination and ships no generated content. See [packaging.md](packaging.md) for why the generator runs at runtime rather than at build time.

**The adapter is integration glue, not a tool.** The compositor adapter ships in `gridline-session` because that package owns integration. Putting it in `gridline-tools` would force the diagnostic commands to carry compositor knowledge, which `specs/compositor-abstraction/spec.md` forbids.

**The portal mapping is configuration, not session logic.** The interface-to-backend mapping is a shipped default, so `gridline-config` owns it, even though the session is what consumes it. `gridline-portals.conf` follows the mapping into `gridline-config` for the same reason, and it installs to `/usr/share/xdg-desktop-portal/gridline-portals.conf`, lower-cased, because the frontend appends `xdg-desktop-portal/` to each base directory it searches and case-folds the desktop name before building the filename. `/etc/xdg` without that subdirectory is not a search path; `/etc/xdg-desktop-portal/` is, and it is reserved for administrator overrides rather than package content. It is a new file under a Gridline-specific name, so it owns no path another package claims.

## What no package does

These apply to every package above, and come straight from `specs/package-delivery/spec.md`:

- **Nothing is fetched at install time.** No maintainer script downloads code or configuration to finish its own installation.
- **No file is claimed twice.** The packages install together without file conflicts, and this is checked at build time rather than discovered at install time.
- **Local changes survive upgrade.** Defaults an administrator is expected to edit are dpkg conffiles.
- **Removal returns the system to stock.** After removing every Gridline package, no session entry remains, GNOME works, and nothing Gridline owned is left behind apart from administrator-modified configuration.

## On `gridline-desktop`

The name is reserved by the `establish-gridline-vision` change and **no `gridline-desktop` package is produced by it**. The metapackage is delivered later, when the component set it would depend on is stable, which no currently scheduled change reaches. It is listed in the deferred requirements table in that change's `proposal.md` for exactly this reason.

When it does ship, it carries no executable, configuration or data files of its own beyond packaging metadata. A metapackage that acquires content has stopped being a metapackage.

## Archive packages Gridline depends on

Gridline ships almost no code of its own: the binaries that make a Gridline session work come from the Ubuntu archive unmodified. The versions published for the Ubuntu 26.04 LTS (`resolute`) baseline are below, with the archive component, because `main` carries a Canonical security-update commitment and `universe` does not.

| Source package | Version on the baseline | Component | Role |
|---|---|---|---|
| `hyprland` | 0.53.3+ds-4 | **universe** | Compositor |
| `hyprlock` | 0.9.2-1build4 | **universe** | Screen locker candidate |
| `hypridle` | 0.1.7-2 | **universe** | Idle daemon candidate |
| `hyprpaper` | 0.8.3-1 | **universe** | Wallpaper candidate |
| `hyprpolkitagent` | 0.1.3-1 | **universe** | polkit agent candidate |
| `xdg-desktop-portal-hyprland` | 1.3.11-1build2 | **universe** | Compositor portal backend |
| `xdg-desktop-portal` | 1.21.1+ds-1ubuntu3.1 | main | Portal frontend |
| `xdg-desktop-portal-gtk` | 1.15.3-2ubuntu1 | main | GTK portal backend |
| `xwayland` | 2:24.1.10-1 | main | X11 application support |
| `gdm3` | 50.1-0ubuntu0.1 | main | Display manager |

Confirming these against the archive is owned by `define-supported-ubuntu-integration`, which also owns re-checking them, since they move with the archive.

**Every Hypr-ecosystem package is in universe.** Two consequences follow and neither is a packaging detail. A security issue in the compositor has no Canonical update commitment behind it, which is a fact users of this session are entitled to know rather than discover. And `gridline-desktop`, once it ships, depends on universe packages and therefore cannot itself live in main — which makes this the central obstacle to any future official-flavor candidacy, not a footnote to it.

The archive also tracks upstream at a distance: `hyprland` 0.53.3 against upstream 0.56.2 released 2026-08-05, three minor releases back. For an audience that follows the Hyprland changelog this is the most visible gap in the project, and Gridline has no mechanism to close it without abandoning the archive-only delivery rule. Stating the baseline version plainly is the honest option.

## Source packages

All four binary packages are built from one source package at `packages/gridline/`. That decision, and the room the layout leaves for additional source packages, is recorded in [packages/gridline/README.md](../packages/gridline/README.md).
