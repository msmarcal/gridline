# Package matrix

The binary packages Gridline is delivered as, what each owns, and which capability requires it.

The authoritative form of these boundaries is `openspec/specs/package-delivery/spec.md`. This page is the readable summary; where the two disagree, the spec wins.

## Packages

| Package | Owns | Required by capability | Status |
|---|---|---|---|
| `gridline-session` | The GDM session entry at `/usr/share/wayland-sessions/gridline.desktop`, the session launcher at `/usr/libexec/gridline/gridline-session`, the session's systemd user target at `/usr/lib/systemd/user/gridline-session.target`, the core integration glue, and the compositor adapter layer | `desktop-session` (the session exists and starts), `platform-integration` (the session reaches the desktop stack), `compositor-abstraction` (the adapter lives here) | **Entry and launcher shipped**; glue and adapter planned |
| `gridline-config` | Shipped defaults under `/etc/xdg`, default keybindings, default themes, the portal interface-to-backend mapping, `gridline-portals.conf`, the administrator's policy input, the lock declarations, and the destination of the generated configuration | `desktop-session` (session-scoped configuration), `platform-integration` (deterministic portal backend selection), `session-manageability` (policy input and locks) | Planned |
| `gridline-tools` | The `gridline` command, `gridline doctor`, the theme and diagnostic commands, and the generator that produces the effective configuration | `platform-integration` (capability gaps are diagnosable), `compositor-abstraction` (the command surface stays compositor-neutral), `session-manageability` (the generator) | Planned |
| `gridline-desktop` | Nothing. A metapackage that pulls in the packages making up a complete Gridline desktop | `package-delivery` (the name is reserved) | **Reserved, not delivered** |

**The package descriptions and this page agree.** `gridline-session` described itself as shipping no content yet until `add-hyprland-session-baseline` corrected it, and the descriptions for `gridline-config` and `gridline-tools` still say so accurately, since those two ship directories and no files. A reader comparing this page's Status column against `dpkg-deb -f <package> Description` should find the two tell the same story; if they do not, the package is the measurement and this page is the one to fix.

Every binary package uses the `gridline-` prefix. See [packages/gridline/README.md](../packages/gridline/README.md) for why the prefix is a convention the build tooling relies on rather than a naming preference.

## Boundaries that are easy to get wrong

Three boundaries sit close enough together to be worth stating explicitly, because `specs/package-delivery/spec.md` requires that no file be claimed by two packages.

**Themes are split by kind, not by subject.** Theme *data* is shipped defaults, so it belongs to `gridline-config`. The theme *commands* that apply or list those themes are part of the command surface, so they belong to `gridline-tools`. A theme feature therefore touches two packages, and neither owns the other's files.

**The generated configuration is split the same way.** `gridline-config` owns the destination, and `gridline-tools` owns the generator that writes into it, because the destination is configuration and the generator is a command. `gridline-config` installed on its own owns an empty destination and ships no generated content. See [packaging.md](packaging.md) for why the generator runs at runtime rather than at build time.

**The adapter is integration glue, not a tool.** The compositor adapter ships in `gridline-session` because that package owns integration. Putting it in `gridline-tools` would force the diagnostic commands to carry compositor knowledge, which `specs/compositor-abstraction/spec.md` forbids.

**Compositor configuration is owned by one package and authored in another's source tree, and that is not a contradiction.** The compositor's defaults and the loader at its search path install through `gridline-config`, which owns shipped defaults under `/etc/xdg`, and are authored under `src/adapters/<compositor>/`, which is where `compositor-abstraction` requires compositor-specific content to live.

Those two rules look like they point at different places for one file, and the reason they do not is that they are about different things: `package-delivery` says which **binary package** owns an installed path, `compositor-abstraction` says where in the **source tree** compositor-specific content may live. The `.install` line that connects them sits in `debian/`, which the isolation check excludes because packaging metadata has to name what the project depends on.

The test this has to pass is the one `compositor-abstraction` cares about: supporting a second compositor means adding an adapter, not editing a file that knows a compositor's name. A second compositor brings its own defaults and its own loader inside its own adapter directory, and nothing outside changes.

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

Gridline ships almost no code of its own: the binaries that make a Gridline session work come from the Ubuntu archive unmodified. The versions published for the Ubuntu 26.04 LTS (`resolute`) baseline are below, with the archive component. Universe carries **no Canonical security-update commitment**. Main does.

| Source package | Version on the baseline | Component | Role |
|---|---|---|---|
| `hyprland` | 0.53.3+ds-4 | **universe** | Compositor |
| `swaylock` | 1.8.4-1 | **universe** | Screen locker |
| `swayidle` | 1.9.0-1 | **universe** | Idle daemon |
| `hyprpaper` | 0.8.3-1 | **universe** | Wallpaper candidate |
| `hyprpolkitagent` | 0.1.3-1 | **universe** | polkit authentication agent |
| `xdg-desktop-portal-hyprland` | 1.3.11-1build2 | **universe** | Compositor portal backend |
| `xdg-desktop-portal` | 1.21.1+ds-1ubuntu3.1 | main | Portal frontend |
| `xdg-desktop-portal-gtk` | 1.15.3-2ubuntu1 | main | GTK portal backend |
| `xwayland` | 2:24.1.10-1 | main | X11 application support |
| `gdm3` | 50.1-0ubuntu0.1 | main | Display manager |
| `ubuntu-session` | 50.1-0ubuntu0.1 | main | The stock session that stays selectable |
| `network-manager` | 1.54.3-2ubuntu3.1 | main | Networking |
| `pipewire` | 1.6.2-1ubuntu1.2 | main | Audio server |
| `wireplumber` | 0.5.13-1ubuntu1.1 | main | Audio session manager |
| `bluez` | 5.85-4ubuntu0.4 | main | Bluetooth |
| `upower` | 1.91.1-1 | main | Battery and power source observability |
| `snapd` | 2.77.1+ubuntu26.04.2 | main | Snap applications |
| `policykit-1` | 127-2ubuntu1.1 | main | polkit daemon, binary `polkitd` |
| `systemd` | 259.9-0ubuntu0.1 | main | Seat, session and power event handling |
| `apparmor` | 5.0.2-0ubuntu1~26.04.1 | main | Confinement |
| `apt` | 3.2.0 | main | Update path |
| `unattended-upgrades` | 2.12ubuntu9 | main | Update path |
| `wayland-utils` | 1.3.0-1 | **universe** | Wayland registry query, used by the compositor adapter |
| `xdg-terminal-exec` | 0.14.0-1 | main | Launches the terminal the system is configured to use |
| `foot` | 1.25.0-1 | **universe** | Terminal emulator, declared because the baseline guarantees none |

### How these were measured

Measured 2026-10-26 against a system running the supported baseline, Ubuntu 26.04.1 LTS amd64, with `main`, `universe`, `restricted` and `multiverse` enabled. Both commands are part of a stock Ubuntu system and neither needs a Gridline package installed:

```
apt-get update
apt-cache policy <package>     # whether it exists, and the candidate version
apt-cache madison <package>    # the archive component that provides it
```

The two answer different questions and are worth running together: `policy` reports what would be installed, `madison` reports which component it comes from. A reader repeating this needs nothing the table does not already name.

These rows were confirmed against the archive by `define-supported-ubuntu-integration`. Re-checking them is not owned by a single change, because they move with the archive rather than with the project: any change that adds, removes or alters a dependency re-checks the rows it touches, using the procedure above. If an automated check is ever built, `add-ci-vm-smoke-tests` is where it belongs, since that is the change that owns running things against a baseline machine.

**[component-ownership.md](component-ownership.md) is authoritative for component facts.** Where that page and this one carry the same version or archive component, it wins, because it is the one organized by component and it carries what each component's absence does. This table is the packaging-side view of the same measurement.

**One row is a candidate rather than a dependency**, marked as such in the Role column: `hyprpaper` for the wallpaper. No requirement rests on it, so it is in no package's `Depends`, and the change that adds the wallpaper owns the choice.

`swayidle` is a dependency again, declared by `add-gridline-configuration-defaults`, which is the change that gave it something to do: it runs the session's lock entry point on inactivity, on the logind lock signal and before suspend. It was a dependency for the length of an earlier change, removed when the blocking components were counted and no idle daemon was among them, and declared again here for a different reason. It is still not a blocking component, and `component-ownership.md` records how the session holds it running without treating its absence as grounds to refuse a session.

**Four universe packages are not Hypr-ecosystem**: `swaylock` and `swayidle` from swaywm, `wayland-utils` from freedesktop, and `foot`. They carry the same universe gap and none of them carries it because of the compositor. Gridline depends on them for reasons of its own: the adapter needs the registry query because the compositor publishes its protocols there and offers no command that lists them, the locker and idle daemon were chosen for the libraries they do **not** link, and the terminal is declared because the baseline supplies none.

This paragraph said "one" and named only `wayland-utils` until `add-gridline-configuration-defaults` counted the rows. It had been wrong since `add-session-lifecycle-integration` declared the locker and the idle daemon, and that change's own coherence pass did not catch it, which is the ordinary way a count in prose drifts from the table above it.

**Every Hypr-ecosystem package is in universe.** Two consequences follow and neither is a packaging detail. A security issue in the compositor has no Canonical update commitment behind it, which is a fact users of this session are entitled to know rather than discover. And `gridline-desktop`, once it ships, depends on universe packages and therefore cannot itself live in main - which makes this the central obstacle to any future official-flavor candidacy, not a footnote to it.

The archive also tracks upstream at a distance: `hyprland` 0.53.3 against upstream 0.56.2 released 2026-08-05, three minor releases back. For an audience that follows the Hyprland changelog this is the most visible gap in the project, and Gridline has no mechanism to close it without abandoning the archive-only delivery rule. Stating the baseline version plainly is the honest option.

## Source packages

All four binary packages are built from one source package at `packages/gridline/`. That decision, and the room the layout leaves for additional source packages, is recorded in [packages/gridline/README.md](../packages/gridline/README.md).
