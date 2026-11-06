# Design

## Context

See proposal.md for motivation. Four measurements against the archive shape this, and one open question the project has avoided answering twice.

**What the archive carries**, read with `apt-cache madison` and `apt-cache show` on the baseline:

| Element | Candidate | Installed size | Hyprland libraries |
|---|---|---|---|
| Status bar | `waybar` 0.15.0-1 | 2400 KB | none |
| Notifications | `mako-notifier` 1.10.0-1build1 | 191 KB | none |
| Wallpaper | `swaybg` 1.2.1-1build1 | 53 KB | none |
| Wallpaper | `hyprpaper` 0.8.3-1 | 495 KB | `libhyprlang2`, `libhyprutils10` |

All in `universe`, which is the gap this project already records for everything it depends on except the display stack.

**The notification daemon was written off and is available.** `docs/user-experience.md` lists `mako` among packages absent from the archive. The archive carries it as `mako-notifier`, a different binary package name for the same component, described as a lightweight notification daemon for Wayland compositors. The other four names on that line are genuinely absent, checked by name and by near-name search, so the correction is one entry rather than a reason to distrust the table.

**The three start in three different ways.** `waybar` and `mako-notifier` each ship systemd user units; `swaybg` ships none, no IPC, no reload signal and no configuration file at all, taking everything including per-output appearance as command-line arguments. Whether the two shipped units are *enabled* is the question `add-session-lifecycle-integration` learned to ask, and this change asks it again rather than assuming.

## Goals / Non-Goals

**Goals:**

- A first login that shows the state of the machine, delivers notifications, and has a background.
- One start mechanism covering three components that the archive packages differently.
- Settle the boundary of the rule that rejected two components for what they link.

**Non-Goals:**

- The launcher and the command menu. They are one piece of work and this is not it.
- Theming. This ships a default appearance; a mechanism that restyles components together is a separate change the target table already separates out.
- Making any of these blocking. A desktop without a bar is a worse desktop, not an unsafe one.

## Decisions

### `swaybg`, and the rule that chose it now has a stated boundary

`add-session-lifecycle-integration` rejected `hyprlock` and `hypridle` for linking `libhyprlang`, the compositor's own configuration language, whose `findConfig` returns the first file it finds and stops. The reason given was specific to where those components sit: building the fail-safe lock path on the configuration model the project had already identified as its central obstacle would be a choice rather than an inheritance.

`hyprpaper` links the same library. A wallpaper daemon is not in the fail-safe path, so the recorded reason does not transfer by itself, and the project has applied this test twice without ever saying how far it reaches. This change says: **the test applies generally, and the fail-safe framing was the strongest case for it rather than its limit.** A component that resolves configuration by first-file-wins is one the manageability capability cannot make policy over, and that is true of a wallpaper as much as of a locker. The difference is how much it costs when it bites, not whether it bites.

On this occasion the general rule and the ordinary engineering answer agree, which is worth noting because it makes the decision cheap rather than brave: `swaybg` is 53 KB against 495 KB and links nothing. What `hyprpaper` offers over it is IPC through `hyprctl` and per-monitor wallpaper handling, and nothing currently specified needs either. Per-output appearance is available in `swaybg` as command-line options.

**The cost to name:** `swaybg` has no IPC and no reload, so changing the wallpaper means restarting the process. The theme change that will restyle components together therefore restarts this one rather than signalling it. At 53 KB that is cheap, and it is the same shape as the idle daemon, whose settings are also arguments in a Gridline-owned unit rather than a file.

### Three components, three packagings, one mechanism

`gridline-session.target` already distinguishes blocking from non-blocking by the systemd vocabulary: `Wants=` for a unit whose absence refuses the session, `Upholds=` for one that is kept running and restarted without being required. systemd reports the two separately, which is what lets the launcher's blocking check read one and ignore the other. All three components here belong under `Upholds=`.

That leaves what each `Upholds=` points at:

- For the two that ship units, the shipped unit, **if** it is suitable. `add-session-lifecycle-integration` found an archive unit shipped disabled with an inert `[Install]` section, and another that was D-Bus activated and needed nothing. Both assumptions were wrong at different times, so each unit here is read before being relied on.
- For `swaybg`, a Gridline-authored unit, since there is nothing to point at. Its configuration is arguments in that unit, exactly as the idle daemon's is.

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
- **Everything added here runs in a plain compositor session too.** → The defaults live at the compositor's system configuration path, which any session of that compositor reads, so an `exec-once` added here starts Gridline's components in a session that is not Gridline's. This is recorded in `docs/compositor-adapter.md` as an open finding and it gets materially worse with this change, since it turns inert keybindings into running processes. Whether that finding is closed before this change lands is a sequencing decision rather than a design one, and it is named here so it is not discovered afterwards.
- **A notification daemon changes what applications can do to the user's screen.** → It is the component's own default behaviour, unmodified, and nothing here grants an application anything it could not already ask for.

## Open Questions

- Whether the two shipped units are enabled, which decides how much the session target has to declare. Measurable before implementation and deliberately not assumed here.
- Whether the leak into a plain compositor session is closed first. The answer changes nothing in this design and changes what a user of the other session experiences.
