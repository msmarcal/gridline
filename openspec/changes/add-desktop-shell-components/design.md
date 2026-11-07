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

### Gridline ships the choice of wallpaper, not an image

The scenario asks for a background and the project ships no image, which this design did not notice it had to settle. The component offers no solid-colour directive, read from the source of the version the archive ships, so the alternatives were shipping an image or naming one.

It names `/usr/share/backgrounds/ubuntu-default-greyscale-wallpaper.png`, a symlink `ubuntu-wallpapers` maintains at the release's own wallpaper, and declares that package. Every other file in that directory is renamed per release and this one is not, so the desktop does not go blank at the next release. The package is in `main`, which is why the long description's list of exceptions to the universe gap grew.

Authoring an image was the alternative and is a separate kind of work: it is Gridline's visual identity rather than its integration, and it belongs with theming, which this change does not do.

**The shipped file is written in the 0.8.3 configuration format, which is not the one its documentation shows.** `preload` is gone and `wallpaper` became a category with a block. The format was read from the source of the archive's version, because a file in the older syntax parses without complaint and displays nothing, which is indistinguishable at runtime from no configuration at all.

### Three components, one mechanism, three units to read first

`gridline-session.target` already distinguishes blocking from non-blocking by the systemd vocabulary: `Wants=` for a unit whose absence refuses the session, `Upholds=` for one that is kept running and restarted without being required. systemd reports the two separately, which is what lets the launcher's blocking check read one and ignore the other. All three components here belong under `Upholds=`.

That leaves what each `Upholds=` points at. All three ship a unit, so in principle all three are pointed at theirs, and in practice each is read first: a shipped unit can be inert, can be activated by something else, or can carry an `ExecStart` that does not take the configuration path this change needs it to take. Measured: none of the three passes a configuration path, so all three need one supplied.

**The configuration path is not named in the drop-in, and this is the decision the section above would otherwise have got wrong.** A drop-in rewrites `ExecStart` for every session on the machine rather than for Gridline's, so naming the path there would make Gridline's defaults reach these components wherever they are started. That is the leak `confine-gridline-configuration` closed for the compositor, reopened in a second place.

So each drop-in routes `ExecStart` through `/usr/libexec/gridline/gridline-shell <role>`, a Gridline program that decides at start time against two questions: is this a Gridline session, read from `XDG_CURRENT_DESKTOP`, and would the component find configuration of its own. Outside Gridline it execs the component with no arguments, so the archive's unit behaves as the archive ships it.

Three consequences worth stating, because each was learned by building it:

- **Which search bases count is narrower than each component's own search.** A system base counts only where the component's package ships nothing into it, so that a file being there means somebody put it there. The bar ships `/etc/xdg/waybar/config.jsonc`; treating that as a user's choice made Gridline's bar configuration unreachable on every machine. What it gives up is that an administrator cannot override Gridline's bar by editing the archive's file, because nothing can tell that edit from the one dpkg installed.
- **The session key has to be removed at teardown.** The user manager outlives a session, so the launcher now unsets `XDG_CURRENT_DESKTOP` as it already unset `WAYLAND_DISPLAY` and `GRIDLINE_CONFIG`.
- **The archive's own packaging undoes the target's purpose.** Each of the three enables its unit from its postinst, into `graphical-session.target.wants`, which belongs to every graphical session. That is the same thing this target exists to avoid, arriving through a dependency. `gridline-session` disables them and records which ones it turned off, and this costs nothing because `Upholds=` starts a unit whatever its enabled state.

**The ordering trap is already known.** A target with `DefaultDependencies=yes` gains an implicit `After=` on everything in `Wants=` and, measured, in `Upholds=` too. Adding three more entries adds three more implicit orderings against units that are themselves ordered after `graphical-session.target`. The target carries no `Before=graphical-session.target` precisely because that closed a cycle once, and this change verifies no cycle appears rather than trusting that the earlier fix covers a case with four entries instead of one.

### Styling is shipped, and the line to theming is where the mechanism starts

A bar with no stylesheet renders as an unstyled strip, which fails the requirement this change adds as surely as having no bar. So a stylesheet ships.

The line is not how much styling, it is whether anything generalises it. This change ships one appearance per component, written as that component's own configuration. It ships no palette shared between them, no named theme, no mechanism that rewrites several components from one source. The moment a colour is defined once and consumed twice, this has become the theme mechanism, and that belongs to the change the target table already separates out.

Stated as a test rather than an intention: if a later theme change can replace these files wholesale without unpicking anything, the line held.

### Where each piece of configuration lives

The split `add-gridline-configuration-defaults` established applies unchanged. Compositor-side configuration is written under the adapter path because it is the compositor's vocabulary. The components' own configuration and styling are their own formats, understood by programs that are not the compositor, so they live outside the adapter. Both install through `gridline-config`, which owns shipped defaults.

**This section used to say the compositor-side part launches these components, and that contradicted the start mechanism decided above.** The session target starts them; the compositor starts none of them. What is genuinely compositor vocabulary turns out to be one thing, which is smaller than the sentence implied: the bar's workspace module, whose name is a compositor's name. Geometry needs nothing, because the bar and the notification daemon are layer-shell clients and the compositor honours the exclusive zone they ask for without being configured to.

That smallness is worth stating rather than filling. A reader expecting an adapter file per component will find one file, and the reason is that the split is about vocabulary rather than about volume.

The isolation check now matches file names as well as contents, so a file named for the compositor outside the adapter path fails the build. That is the check doing its job here rather than an obstacle: a bar's stylesheet is not compositor-specific and should not be named as though it were.

## Risks / Trade-offs

- **Three more `universe` dependencies, one of them 2400 KB.** → The bar is the largest single component the project will have declared, and it is the one element of the target table that cannot be approximated. The size is recorded rather than defended.
- **Everything added here runs in a plain compositor session too.** → **Closed before this change reached implementation, and by two mechanisms rather than the one route this risk expected.** It assumed the components would be started by an `exec-once` in defaults living at the compositor's own system configuration path, which every session of that compositor reads. `confine-gridline-configuration` moved those defaults to a path the session names, and the session-entry diversion stopped the compositor's entry being offered at the greeter. The start mechanism decided above removes the premise as well: the session target starts these components, and no session but Gridline's starts that target. The scenario still has to be measured against what this change ships, which is group 5's remaining work, but it is no longer a risk with an open route.
- **A notification daemon changes what applications can do to the user's screen.** → It is the component's own default behaviour, unmodified, and nothing here grants an application anything it could not already ask for.

## Open Questions

Both questions below were answered by measuring during implementation, and the answers are kept here with them rather than only in `tasks.md`, because this file is the one a later reader opens to find out why.

- ~~Whether each of the three shipped units is enabled, and whether the wallpaper's passes an explicit configuration path.~~ **Answered, and the first half came out the opposite way from the reading that preceded it.** No package ships an enable symlink in its `.deb`, which is a different question from whether the unit ends up enabled: each postinst runs `deb-systemd-helper --user enable`, so all three are enabled into every graphical session, and `gridline-session` now disables them. None of the three passes a configuration path, the wallpaper's included, so all three take a drop-in.
- ~~Which route closes the leak into a plain compositor session.~~ **Answered by three mechanisms rather than the one route expected, and none of them is this change's.** `confine-gridline-configuration` moved the compositor's defaults to a path the session names, the session-entry diversion stopped the compositor's entry being offered, and the start mechanism decided above means the components are started by a target only Gridline's session starts. This change verifies rather than builds, which is what the question anticipated.

Still open, and recorded rather than closed by argument:

- **Whether a component that cannot start on a given machine should be kept running forever.** `Upholds=` with `Restart=on-failure` revives a unit past its own start limit, measured at 59 starts in two minutes on the verification VM, where the wallpaper daemon cannot allocate buffers. The cause there is that machine's graphics and not the component, confirmed on hardware that renders, so nothing here is evidence for changing the restart policy. The question is whether the mechanism should bound itself anyway.
