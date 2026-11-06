# Design

## Context

See proposal.md for motivation. Four measurements against the archive shape this, and one open question the project has avoided answering twice.

**What the archive carries**, read with `apt-cache madison` and `apt-cache show` on the baseline:

| Element | Candidate | Installed size | Hyprland libraries |
|---|---|---|---|
| Status bar | `waybar` 0.15.0-1 | 2400 KB | none |
| Notifications | `mako-notifier` 1.10.0-1build1 | 191 KB | none |
| Wallpaper | `hyprpaper` 0.8.3-1 | 495 KB | `libhyprlang2`, `libhyprutils10` |
| Wallpaper | `swaybg` 1.2.1-1build1 | 53 KB | none |

All in `universe`, which is the gap this project already records for everything it depends on except the display stack.

**The notification daemon was written off and is available.** `docs/user-experience.md` lists `mako` among packages absent from the archive. The archive carries it as `mako-notifier`, a different binary package name for the same component, described as a lightweight notification daemon for Wayland compositors. The other four names on that line are genuinely absent, checked by name and by near-name search, so the correction is one entry rather than a reason to distrust the table.

**All three ship systemd user units.** `waybar`, `mako-notifier` and `hyprpaper` each carry one. Whether a shipped unit is *enabled* is the question `add-session-lifecycle-integration` learned to ask after finding one disabled with an inert `[Install]` section and another needing nothing because it was D-Bus activated. Those were opposite answers to the same question, so this change measures each of the three rather than assuming any of them.

## Goals / Non-Goals

**Goals:**

- A first login that shows the state of the machine, delivers notifications, and has a background.
- One start mechanism covering all three, with each shipped unit read rather than trusted.
- Say what the rule that rejected two earlier components actually tests, since this change selects a third that trips the same surface reading of it.

**Non-Goals:**

- The launcher and the command menu. They are one piece of work and this is not it.
- Theming. This ships a default appearance; a mechanism that restyles components together is a separate change the target table already separates out.
- Making any of these blocking. A desktop without a bar is a worse desktop, not an unsafe one.

## Decisions

### `hyprpaper`, and the rule that rejected its siblings is about the search, not the library

`add-session-lifecycle-integration` rejected `hyprlock` and `hypridle` for linking `libhyprlang`, the compositor's own configuration language, whose `findConfig` returns the first file it finds and stops. `hyprpaper` links the same library, so choosing it has to say what that earlier test actually tests.

**It tests the search, and the search is avoidable.** The harm recorded against `findConfig` is that a user file replaces an administrator file entirely and silently, which is what puts the setting beyond the reach of the manageability capability. That happens because the component searches. `hyprpaper` takes `-c`/`--config`, and a named file is not searched for: Gridline's unit passes the path, `findConfig` never runs its search, and the failure mode the earlier change was avoiding does not arise.

Framing it as a rule about the library would have been the wrong generalisation, and it is worth saying so plainly because an earlier draft of this design made exactly that mistake. A project that rejects components for what they link rejects them for a property that says nothing on its own; a project that rejects uncontrolled configuration search rejects the thing that actually bites.

**What this costs against `swaybg`**, which was the alternative and is not a bad one: 495 KB and two libraries against 53 KB and none. What it buys: a systemd user unit the archive already ships, where `swaybg` ships none and Gridline would author one; a configuration file rather than only command-line arguments; and wallpaper changes without restarting the process, which the theme change will want when it restyles components together.

**A finding about an archived decision, recorded rather than acted on.** `hyprlock`'s binary carries `--config`, measured by inspecting it. So the same escape was available for the locker, and the rejection recorded against it rested on an argument weaker than it appeared: with an explicit path the manageability position would have been *stronger*, since a user could not replace a policy file the unit names. `hypridle` carries no such flag, so its rejection stands on its own. Nothing here reopens the locker: it is selected, shipped and verified. The reason to write this down is that the next component choice will meet the same argument, and the project should meet it with the sharper version.

### Three components, one mechanism, three units to read first

`gridline-session.target` already distinguishes blocking from non-blocking by the systemd vocabulary: `Wants=` for a unit whose absence refuses the session, `Upholds=` for one that is kept running and restarted without being required. systemd reports the two separately, which is what lets the launcher's blocking check read one and ignore the other. All three components here belong under `Upholds=`.

That leaves what each `Upholds=` points at. All three ship a unit, so in principle all three are pointed at theirs, and in practice each is read first: a shipped unit can be inert, can be activated by something else, or can carry an `ExecStart` that does not take the configuration path this change needs it to take. The wallpaper's in particular has to pass `--config`, which means either the shipped unit already does or Gridline supplies a drop-in that makes it.

**The ordering trap is already known.** A target with `DefaultDependencies=yes` gains an implicit `After=` on everything in `Wants=` and, measured, in `Upholds=` too. Adding three more entries adds three more implicit orderings against units that are themselves ordered after `graphical-session.target`. The target carries no `Before=graphical-session.target` precisely because that closed a cycle once, and this change verifies no cycle appears rather than trusting that the earlier fix covers a case with four entries instead of one.

### Styling is shipped, and the line to theming is where the mechanism starts

A bar with no stylesheet renders as an unstyled strip, which fails the requirement this change adds as surely as having no bar. So a stylesheet ships.

The line is not how much styling, it is whether anything generalises it. This change ships one appearance per component, written as that component's own configuration. It ships no palette shared between them, no named theme, no mechanism that rewrites several components from one source. The moment a colour is defined once and consumed twice, this has become the theme mechanism, and that belongs to the change the target table already separates out.

Stated as a test rather than an intention: if a later theme change can replace these files wholesale without unpicking anything, the line held.

### Where each piece of configuration lives

The split `add-gridline-configuration-defaults` established applies unchanged. Compositor-side configuration, which is the part that launches these components and places them, is written under the adapter path because it is the compositor's vocabulary. The components' own configuration and styling are their own formats, understood by programs that are not the compositor, so they live outside the adapter. Both install through `gridline-config`, which owns shipped defaults.

The isolation check now matches file names as well as contents, so a file named for the compositor outside the adapter path fails the build. That is the check doing its job here rather than an obstacle: a bar's stylesheet is not compositor-specific and should not be named as though it were.

## Risks / Trade-offs

- **Three more `universe` dependencies, one of them 2400 KB.** → The bar is the largest single component the project will have declared, and it is the one element of the target table that cannot be approximated. The size is recorded rather than defended.
- **Everything added here runs in a plain compositor session too.** → The defaults live at the compositor's system configuration path, which any session of that compositor reads, so an `exec-once` added here starts Gridline's components in a session that is not Gridline's. This is recorded in `docs/compositor-adapter.md` as an open finding and it gets materially worse with this change, since it turns inert keybindings into running processes. **The delta settles what to do about it**: a scenario requires that no session other than Gridline start these components, so closing it is work this change owns rather than an order of play it can choose. Which of the two available routes closes it, the configuration reaching only the Gridline session or the other session entry no longer being offered, is open; whether it closes is not.
- **A notification daemon changes what applications can do to the user's screen.** → It is the component's own default behaviour, unmodified, and nothing here grants an application anything it could not already ask for.

## Open Questions

- Whether each of the three shipped units is enabled, and whether the wallpaper's passes an explicit configuration path. Both decide how much the session target and any drop-in have to declare. Measurable before implementation and deliberately not assumed here.
- Which route closes the leak into a plain compositor session. Not whether: a delta scenario requires it, so it is in scope. The two routes are making the compositor configuration reach only the Gridline session and diverting the other session entry, the second of which `docs/package-matrix.md` already records as decided for reasons of its own. If that lands first, this change inherits it and verifies rather than builds.
