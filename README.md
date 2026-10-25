# Project Gridline

An optional Wayland tiling desktop session for Ubuntu, delivered as Debian
packages and installed on top of a stock Ubuntu system.

Select Gridline at the login screen to get a tiling desktop. Select Ubuntu to
get GNOME. Remove the packages and the machine is exactly as it was.

> **Status: specification. There is nothing to install yet.**
> The repository currently contains documentation and specifications. No
> package has been built, and no code has been written. See
> [Project status](#project-status) for what exists and what comes next.

## What makes it different

Not tiling, and not polish. Tiling sessions for Linux already exist, and
projects on rolling bases ship them with a faster cadence and a stronger
aesthetic than Gridline will have.

**Gridline is a third option.** You use Ubuntu and you like using it. You work
in the terminal. You saw Omarchy and wanted that. Today your choices are to
leave Ubuntu for a rolling base, or to assemble the stack by hand and maintain
it yourself forever. Gridline is the Omarchy-like experience delivered as apt
packages on the Ubuntu you already run, with the Ubuntu session sitting intact
beside it in case you want out.

The audience is small, the way Omarchy's is. That is the nature of the thing
rather than a problem to solve.

The rigor elsewhere in this project follows from that, and not from any
compliance requirement: packages instead of install scripts, GNOME always
present, clean removal, and the compositor kept behind a single boundary. You
are going to run this on the machine you work on, and an experimental desktop
should not be able to leave you without a computer on a Tuesday morning.

One gap is worth knowing before you install anything. The compositor stack lives
in Ubuntu's `universe` component, which carries no Canonical security-update
commitment, so patching it is the operator's responsibility, which on your own
machine means yours. Gridline documents that instead of hiding it; see
[security-and-manageability.md](docs/security-and-manageability.md).

## Design rules

These are constraints the project holds itself to, not aspirations:

- **Debian packages only.** No install script, no post-install download, no
  manual file copying. If it cannot ship in a package, it does not ship.
- **GNOME stays.** Installed, selectable and supported. Gridline is additive and
  removable, and no change may replace or remove the Ubuntu session.
- **Nothing else's files get edited.** GDM, PAM, NetworkManager, PipeWire and
  polkit are influenced only through named drop-ins under paths Gridline owns.
- **Archive packages are preferred** over third-party sources, so that what is
  installed is what Ubuntu ships.
- **One compositor, behind a boundary.** Hyprland is the initial compositor, but
  compositor-specific logic is isolated behind an adapter in a single location.

## The experience

Modelled on Omarchy, which showed that a tiling desktop can be immediately
usable rather than a configuration project.

Omarchy 4.0 replaced its desktop with a single Quickshell process and moved
compositor configuration to Lua. The stack it discarded is the stack Ubuntu
packages; the stack it adopted is not in the archive. Gridline therefore targets
the **Omarchy 3.x experience built entirely from archive packages**: a
Super-centric keymap, a command menu, coordinated theming, capture tools, idle
and lock, a status bar and a launcher.

Three things are excluded on policy rather than availability:

| Excluded | Why |
| --- | --- |
| Driving system updates | Patching is apt's job. Gridline does not interpose itself between you and your package manager. |
| Shortcuts to proprietary services | Gridline binds keys only to software it delivers. The remaining combinations stay unbound and documented. |
| Installing software on demand | Post-install download is prohibited by the delivery rules. A machine that fetches software after install cannot be returned to its prior state by removing packages. |

The full inventory, including what the baseline cannot reach, is in
[user-experience.md](docs/user-experience.md).

## Project status

Development is specification-first using [OpenSpec](https://openspec.dev). Two
changes are complete, both documentation and specifications by design:

| Change | Scope |
| --- | --- |
| `establish-gridline-vision` | The foundation: baseline, session model, delivery rules, four capability specifications |
| `add-manageability-capability` | Central configuration, enforced policy, and the policy layer |

Together they specify 38 requirements across five capabilities. The next change,
`define-debian-package-architecture`, is the first that produces something
installable.

Validate the specifications with:

```
openspec validate --all --strict
```

## Documentation

| Document | Contents |
| --- | --- |
| [vision.md](docs/vision.md) | What Gridline is, the supported baseline, and the non-goals |
| [architecture.md](docs/architecture.md) | The five layers and their boundaries |
| [user-experience.md](docs/user-experience.md) | The experience target, what is reachable, and what is excluded |
| [security-and-manageability.md](docs/security-and-manageability.md) | The facts that constrain the project: the `universe` gap, what policy can and cannot enforce |
| [package-matrix.md](docs/package-matrix.md) | The binary packages and what each owns |
| [packaging.md](docs/packaging.md) | How the packages are built, and how they behave across upgrade and removal |
| [component-ownership.md](docs/component-ownership.md) | Every component, its owner, and its fallback |
| [compositor-adapter.md](docs/compositor-adapter.md) | The adapter contract and the isolation boundary |
| [configuration.md](docs/configuration.md) | Configuration resolution and session scoping |
| [recovery.md](docs/recovery.md) | What a user does when a session fails |

Packaging layout and the source-package decisions are in
[packages/gridline/README.md](packages/gridline/README.md).

## Hardening reference

Hardening is a secondary concern here, not the point of the project. Where a
reference is useful, it is the CIS Benchmark for Ubuntu Linux, **Level 1
Workstation**, together with screen lock policy and disk encryption: a generic
floor whose desktop controls are concrete enough to check against.

FIPS, STIG and PCI are explicitly out of scope. They require formal validation
that a desktop session project does not deliver, and claiming them would be the
kind of overreach that makes everything else here untrustworthy.

## Name

The project is **Project Gridline**; the composed form is used in prose, while
`gridline` remains the command, the source package and the `gridline-` prefix.
"Gridline" is a common word in software and the name was checked rather than
assumed; the findings are recorded in [vision.md](docs/vision.md#the-name).

## Licence

MIT. See [LICENSE](LICENSE).

Attribution is the only condition: keep the copyright notice and the licence
text in copies and in substantial portions. Beyond that, fork it, ship it,
rename it, sell it.

Gridline is a personal project. It is not a Canonical product, it is not
endorsed by Canonical, and nothing here speaks for Canonical.

## Contributing

The specifications come before the code. A change that adds behaviour starts as
an OpenSpec change with requirements and scenarios, not as a patch.

Commits follow [Conventional Commits](https://www.conventionalcommits.org/)
`1.0.0-beta.2`, with the subject line kept to 72 characters.
