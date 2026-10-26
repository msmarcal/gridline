# Design

## Context

See proposal.md for motivation. Three constraints shape the approach.

**The measurement is done.** Every component the twelve `platform-integration` requirements rest on was queried against the baseline archive from a VM running the supported baseline, Ubuntu 26.04.1 LTS amd64, on 2026-10-26. All nineteen exist:

| Package | Version | Archive component |
|---|---|---|
| `gdm3` | 50.1-0ubuntu0.1 | main |
| `ubuntu-session` | 50.1-0ubuntu0.1 | main |
| `network-manager` | 1.54.3-2ubuntu3.1 | main |
| `pipewire` | 1.6.2-1ubuntu1.2 | main |
| `wireplumber` | 0.5.13-1ubuntu1.1 | main |
| `bluez` | 5.85-4ubuntu0.4 | main |
| `upower` | 1.91.1-1 | main |
| `xdg-desktop-portal` | 1.21.1+ds-1ubuntu3.1 | main |
| `xdg-desktop-portal-gtk` | 1.15.3-2ubuntu1 | main |
| `xwayland` | 2:24.1.10-1 | main |
| `snapd` | 2.77.1+ubuntu26.04.2 | main |
| `apparmor` | 5.0.2-0ubuntu1~26.04.1 | main |
| `polkitd` | 127-2ubuntu1.1 | main |
| `xdg-desktop-portal-hyprland` | 1.3.11-1build2 | **universe** |
| `hyprland` | 0.53.3+ds-4 | **universe** |
| `hyprlock` | 0.9.2-1build4 | **universe** |
| `hypridle` | 0.1.7-2 | **universe** |
| `hyprpaper` | 0.8.3-1 | **universe** |
| `hyprpolkitagent` | 0.1.3-1 | **universe** |
| `apt` | 3.2.0 | main |
| `systemd` | 259.9-0ubuntu0.1 | main |
| `unattended-upgrades` | 2.12ubuntu9 | main |

Twenty-two components, sixteen in `main` and six in `universe`, and every `universe` entry is Hypr-ecosystem. The versions confirm what `docs/package-matrix.md` already recorded for the rows it carries.

The last three rows were added after the first draft of this design, which measured nineteen components derived from the `platform-integration` requirements rather than from `docs/component-ownership.md`. Cross-checking the two sets found `apt`, `systemd` and `unattended-upgrades` in the ownership matrix and absent from the measurement. That the two sets both held nineteen entries was coincidence, and a count is not a cross-check.

The cross-check also ran the other way, and what it found there is not a gap in the ownership matrix. Three measured components have no row in it: `hypridle` and `hyprpaper`, which are candidates for an idle daemon and a wallpaper tool that no current requirement rests on, and `ubuntu-session`, which the matrix names in prose as the owner of the session directory rather than as a component Gridline depends on. Adding rows for them would record a dependency the project has not taken. They stay measured and unlisted, and the change that selects an idle daemon is the one that gives `hypridle` a row.

One naming correction falls out, and it has three distinct values rather than two. The ownership matrix names the polkit component `polkit-1`. On this baseline the binary package is `polkitd` and its source package is `policykit-1`, so `polkit-1` is neither. The matrix's column is **Ubuntu source**, so the value that belongs there is `policykit-1`; `polkitd` belongs wherever the binary is what matters.

**This is a definition change.** It produces documentation and specs, never packaging metadata or builds. That boundary is what keeps the classification below a record rather than a half-made decision: saying a component is blocking is this change's work, writing `Depends:` is the work of whichever change delivers a session that can use it.

**The records disagree in one place.** `docs/component-ownership.md` carries absence behavior per component; `docs/package-matrix.md` carries versions and archive components. They overlap on the archive component column, and nothing says which wins.

## Goals / Non-Goals

**Goals:**

- Make the component set a declared, checkable fact rather than prose nobody tests.
- Record archive support status where the component is described, not only in a separate document a reader may not reach.
- Attribute every deferred blocking dependency to the change that owns it.

**Non-Goals:**

- Writing any dependency field. See Context.
- Proving any integration behaves correctly.
- Choosing the locker or the polkit agent, or declaring the compositor.
- Authoring an AppArmor profile. The existing requirement is that Gridline does not weaken confinement, which shipping no profile satisfies.

## Decisions

### The classification is recorded now; the declaration it implies is not

Each component is recorded as blocking or degrading. The degrading set is `network-manager`, `pipewire`, `wireplumber`, `bluez`, `upower`, `xwayland`, `xdg-desktop-portal`, `xdg-desktop-portal-gtk` and `snapd`: all nine in `main`, and a session missing any of them starts with a reported capability gap. The blocking set is the five `docs/architecture.md` already names.

The alternative considered was carrying the classification straight into `debian/control` in this change, which would have satisfied `package-delivery`'s dependency requirement sooner. It was rejected on the project's own rule that a definition change produces documentation and specs only. The rule earns its place here: four of the five blocking components are not yet chosen or deliverable, so a control file written now would encode a decision for `hyprland` that `add-hyprland-session-baseline` has not made.

### Deferred blocking dependencies are attributed, not left blank

Rather than let the four undeclarable blocking components read as oversights, each is recorded against the change that owns it: the compositor and its portal backend to `add-hyprland-session-baseline`, the locker and the polkit agent to `add-session-lifecycle-integration`. The fifth, the compositor adapter, is Gridline's own and already ships in `gridline-session`.

This is the project's established treatment for work that is real but not yet actionable, and it is what distinguishes a deferral from an omission: an omission is silent, a deferral names its owner.

### `docs/component-ownership.md` is authoritative for component facts

Where the two documents overlap, the ownership matrix wins, because it is organized by component and already carries the absence behavior these requirements make binding. `docs/package-matrix.md` keeps its archive table as the packaging-side view and states which document is authoritative, so a future reader who finds a disagreement knows which one to correct.

### Re-checking is a documented procedure, not a script

The new requirement is that recorded facts are verifiable against the archive. This change records the query that produces them and the date of the measurement, rather than shipping a checker. A checker belongs with `add-session-health-check`, which owns the diagnostic surface; building one here would put a tool in the wrong package and duplicate work that change already owns.

### The `universe` gap is restated beside each dependency

The project context already requires the gap be documented rather than hidden. The new requirement makes its presence alongside the dependency a contract rather than a courtesy, because a reader checking what a component costs in support terms is reading the component's row, not a separate security document.

## Risks / Trade-offs

- **The archive moves and the records go stale.** → The requirement makes staleness a detectable defect rather than an invisible one, and the recorded procedure makes checking cheap. Versions carry the measurement date so a reader knows their age.
- **A classification recorded but not declared can drift from what `debian/control` eventually says.** → The requirement that the declaration match the classification is written now and becomes checkable the moment a dependency field exists, so the drift fails a check rather than passing unnoticed.
- **Restating the `universe` gap in more places invites divergence between the copies.** → The gap is one sentence with one meaning. This project has already seen a two-copy claim drift, in the journal mechanism the archived `add-gdm-wayland-session` corrected, which is why the restatement is a requirement rather than an editorial habit.
- **The polkit naming correction can introduce a second error in place of the first.** → The column being corrected decides the value. A source column takes `policykit-1` and a binary reference takes `polkitd`; replacing `polkit-1` with the wrong one of those is not a fix. The correction is made in live documents only, and archived change artifacts keep what they said, which is the project's established rule.

## Migration Plan

No migration. Nothing installed changes, because nothing installed is touched. Rollback is reverting the commit.
