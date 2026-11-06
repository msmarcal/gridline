# Tasks

## 1. The measurements the design deferred

These come first because each one changes how much the session target has to declare. The design names them as open rather than assuming them, after an earlier change got the same question wrong in both directions on two different units.

- [x] 1.1 **None of the three ships an enable symlink**, read from the packages themselves rather than from an installed machine: zero `.wants/` entries in each. All three carry an `[Install] WantedBy=graphical-session.target` that is inert, the same state `hyprpolkitagent` was found in.

  Both unit roots were listed on the machine as the task requires. One entry there is worth recording and is not ours: `foot` enables `foot-server.service` and `.socket` into `/etc/systemd/user/graphical-session.target.wants/`, so a dependency Gridline declared enabled units for every graphical session on the machine, GNOME included. It is that package's own packaging decision and harmless, being socket-activated, but it is a consequence of our dependency rather than of our code
- [x] 1.2 Read from the units, which run a bare binary in all three cases: `/usr/bin/waybar`, `/usr/bin/mako`, `/usr/bin/hyprpaper`. **None passes a configuration path**, so the wallpaper's does not either, which the design left open and assumed would need answering.

  Their shapes differ in a way that matters. `waybar.service` is a plain service. `mako.service` is `Type=dbus` with `BusName=org.freedesktop.Notifications`, and the package ships a matching D-Bus activation file, so it starts on demand when an application sends a notification and needs nothing to want it. `hyprpaper.service` is `Type=simple` with `ConditionEnvironment=WAYLAND_DISPLAY`
- [x] 1.3 **It honours `--config`, and the design's assumption holds.** Measured on the VM against the running compositor, with `strace -e trace=openat` on the daemon rather than its exit code: with the flag it opens the named file outside every path it searches, `openat(AT_FDCWD, "/tmp/gridline-probe/good.conf", O_RDONLY) = 3`; without it, and with nothing in the search path, it opens no configuration file at all. The two cases differ in exactly that one syscall, which is the discriminating pair.

  Three instruments were wrong before that one, and the third found the answer only because the first two had controls. Run headless with no Wayland display, the daemon dies in `XDG_RUNTIME_DIR` with SIGSEGV **before reading any configuration**, and a nonexistent config path produces output identical to a valid one, so the whole first attempt measured nothing. `hyprctl hyprpaper listloaded` answers `Unknown hyprpaper request`: 0.8.x moved its IPC to a Wayland protocol, so the documented observation point no longer exists. And `--verbose` logs nothing about configuration whatsoever: four cases, including an invalid directive, produced logs of 90 lines that are byte-identical once addresses are masked

- [x] 1.6 **The 0.8.3 syntax, read from the source of the archive's own version and then confirmed by running it.** `apt-get source` is unavailable with no `deb-src`, so the orig tarball came from the pool and its sha256 was checked against the `.dsc` before extracting. `src/config/ConfigManager.cpp:55-67` registers the whole surface:

  ```
  splash = 1          splash_offset = 20       splash_opacity = 0.8      ipc = 1

  wallpaper {         # a special category keyed on monitor, one block per wallpaper
      monitor =       # empty applies to every output
      path =          # a file, or a directory, which it walks for up to 1024 images
      fit_mode = cover
      timeout = 0     # non-zero rotates between the images found
  }

  source = /path      # honours ~ and glob patterns, relative to the including file
  ```

  **The finding recorded here before this was wrong, and wrong in a way that would have misled group 3.** It claimed `preload`, `wallpaper`, `splash` and `ipc` do not exist in the binary and concluded the published syntax does not apply at all. The string measurement was right and the inference was not: `splash` and `ipc` are still flat keys, and `wallpaper` still exists but became a category with a block instead of a one-line value. Only `preload` is genuinely gone, its job now done by `path` accepting a directory.

  **What makes this worth a task rather than a footnote is how the wrong syntax fails.** Measured against the running compositor: the 0.8.3 syntax gets past the parse and into allocating a buffer to draw with, while the published 0.7 syntax and a key invented on the spot, `gridline_not_a_key = 1`, behave **identically to each other** - the daemon starts, reports nothing, exits nothing, and idles with no wallpaper for as long as it is left running. `throwAllErrors` does not reject an unknown key, `init()` logs no error, and the exit code is whatever killed it. So a file copied from a current example is indistinguishable at runtime from no configuration at all, which is precisely the failure this project keeps finding. Task 3.1 authors against the list above rather than against an example, and task 6.3 is the only check that can catch getting it wrong, since nothing short of looking at the screen distinguishes a wallpaper configuration that does nothing from one that was never shipped

  Not obtained here: the wallpaper actually displayed. Started outside the session's own cgroup, logind denies the process DRM access and the daemon segfaults in buffer allocation, so the visual confirmation belongs to group 6, which runs in the session from the greeter
- [x] 1.4 Both can be named explicitly: the binary carries `--config` and `--style`. It also searches `$XDG_CONFIG_HOME/waybar/` then `/etc/xdg/waybar/`, so unlike the other two it has a system path Gridline could ship into.

  Its `include` directive has the precedence **inverted** from the compositor's `source`: the manual states that in case of duplicate options the first defined value wins, including file before included file. So a user who includes Gridline's configuration overrides it by writing their own values anywhere in their file, rather than by writing them after the include
- [x] 1.5 It searches `$XDG_CONFIG_HOME/mako/config` and `~/.mako/config` and **no system path at all**, measured against the binary with a fabricated flag as a negative control. It accepts `--config`, and supports an `include` directive taking an absolute path.

  The question this task asked cannot be answered as posed, because there is no system file for a user file to replace or merge with. What it exposes instead is the reverse problem, recorded in the pause below: passing `--config` is the only way to ship configuration for it, and doing so stops the user's own file being read at all

## 2. Selection and declaration

- [x] 2.1 Declared `waybar`, `mako-notifier` and `hyprpaper`, and read back from the built package rather than from the control file: `dpkg-deb -f gridline-session_0.0.1_all.deb Depends` lists twelve entries with the three new ones at 10, 11 and 12.

  **The notification daemon is `mako-notifier`, not `mako`**, which is the same class of error as the terminal row: the archive carries no package named `mako`, so declaring the upstream name would have failed to install
- [x] 2.2 All three resolve from the Ubuntu archive alone, measured together on the VM: **20 URIs, every one on `archive.ubuntu.com`**, ten from the main pool and ten from universe, and 20 newly installed, so the three dependency closures do not overlap.

  | Component | Packages | Download | Installed |
  |---|---|---|---|
  | `waybar` | 16 | 3.4 MiB | 13.5 MiB |
  | `mako-notifier` | 1 | 0.1 MiB | 0.2 MiB |
  | `hyprpaper` | 3 | 0.5 MiB | 1.9 MiB |

  The bar is the largest component the project has declared, as the task anticipated. **The first run of this measurement was wrong and the bench was dirty with my own work**: `hyprpaper` reported zero packages to fetch against three to install, because installing and purging it for task 1.3 had left its `.deb` in the apt cache. The numbers above are from `apt-get clean` first. Two instrument errors as well: `--print-uris` carries the size in field 3, not 2, and `apt-get install -s` prints no "additional disk space" line at all, so the installed size is the sum of `Installed-Size` over the `Inst` list
- [x] 2.3 Stated in the long description, and verified identical rather than similar: the sentence pair extracted from the built package and from `docs/component-ownership.md` compare equal after stripping markdown emphasis and squeezing the whitespace the control file's continuation lines add.

  **The claim it replaced was false, and false before this change.** It read that every package `gridline-session` depends on except `xwayland` is in universe. `xdg-terminal-exec` is in **main**, confirmed from `apt-cache policy` rather than from the `Section` field, so there are two exceptions and the description now names both. `docs/package-matrix.md` already recorded it correctly, so the defect was in the package alone. Also added: a paragraph saying the three draw the desktop and that none of them is blocking, since a reader of the description would otherwise see three new dependencies and no account of them
- [x] 2.4 Recorded in both documents. `docs/component-ownership.md` gains a **Desktop shell** section, separate from Desktop integration because those rows are platform services Gridline surfaces while these are components Gridline selected, with the sizes, the D-Bus activation of the notification daemon, and the archive-only measurement. `docs/package-matrix.md` gains three dependency rows.

  The wallpaper row records what the design settles, in the form that stops it being re-litigated: `hyprlock` and `hypridle` were set aside for linking `libhyprlang2` and `libhyprutils10`, **and this one links them too**. The rule tests the configuration search, not the library, and `hyprpaper` is admitted by accepting `--config`, which task 1.3 measured with `strace` on the running daemon
- [x] 2.5 Corrected. `mako` moved from the absent row to the present-in-universe row, with a paragraph recording that the archive carries it as `mako-notifier` and that the original check searched for the name the project calls itself. The other four were re-checked rather than taken on trust: `walker`, `satty`, `mise` and `impala` resolve to no package, and `quickshell` on the row above does not either, so all five stay.

  Verified by grep across `docs/`, `packages/` and `openspec/specs/`: the only live text pairing that component with "absent" is the correction itself, which records what was wrong rather than repeating it

## 3. Configuration and styling

**The pattern these tasks waited for is settled, and they are no longer paused.** Group 1 found that only the bar has a system configuration path, and then that using it would repeat a violation already shipped: `gridline-config` declared `/etc/xdg/hypr/hyprland.conf`, and `package-delivery` requires every declared path to carry a Gridline-identifying name. No gate caught it, because the ownership gate tests for collisions between Gridline packages rather than for Gridline-named-ness, and the manifest line that declared it sat in `debian/`, outside the isolation check.

`confine-gridline-configuration` closed that and these tasks inherit its result rather than inventing one. The answer for all three is the answer for the compositor: a Gridline-named path, named explicitly when the session starts the component rather than placed where the component searches, and named **only when the component's own search would find nothing**, so that a user's own configuration wins by existing. That last clause is the part worth carrying over, because naming a path unconditionally takes the user's configuration away from them, which is the problem the compositor had to solve and which `mako` has in a sharper form: group 1 measured that it searches no system path at all, so `--config` is the only way to ship configuration for it and passing it stops the user's file being read.

The gate that was missing exists: `tools/check-declared-paths` fails a manifest line whose path carries no project name, and it matches on any segment rather than the whole path, so a Gridline-named file in a directory another component owns still passes.



- [x] 3.1 Authored four files outside the adapter path, under `/etc/xdg/gridline/shell/`, mirroring `compositor/` for the compositor: `bar.jsonc`, `bar-style.css`, `notifications.conf` and `wallpaper.conf`. Isolation check after: **5 files inside `src/adapters`, 15 outside, 0 references found**, with the name match as well as the content match.

  The wallpaper file is the first thing written in the 0.8.3 syntax task 1.6 established, and it carries `splash = 0` for a reason outside this change: the component draws its own project name over the wallpaper by default, and `desktop-session` already requires that a first login present no upstream project messaging.

  **Where the default wallpaper comes from was a decision the plan did not settle**, since the scenario asks for the background Gridline ships and Gridline ships no image. Resolved by depending on `ubuntu-wallpapers` and naming `/usr/share/backgrounds/ubuntu-default-greyscale-wallpaper.png`, a symlink the package ships, confirmed in its file list, and which tracks the release through its dependency on `ubuntu-wallpapers-resolute`. Every other file in that directory is renamed per release; this one is not. The component offers no solid-colour directive, read from the 0.8.3 source, so the alternative was shipping an image, and Gridline ships the choice instead. The package is in **main**, so the long description's exception list grew to three and was corrected again
- [x] 3.2 **One adapter file, and the task's own wording was wrong about why.** It says the compositor-side configuration launches and places these components. It does neither: the session target starts them, as the design decided, and they are layer-shell clients whose exclusive zone the compositor honours without being configured to. The design asserted both things in two places; it now asserts one, and the preamble of group 5 that repeated the `exec-once` claim is corrected too.

  What is genuinely compositor vocabulary is the bar's workspace module, whose name is a compositor's name, so `src/adapters/hyprland/bar.jsonc` carries it and the bar's own configuration includes it. The include's precedence is the one task 1.4 measured, inverted from the compositor's `source`: the including file wins on a duplicate key, so `modules-left` is deliberately absent from the outer file, which is what lets the adapter supply it
- [x] 3.3 Read from the shipped files rather than argued: **two of the five carry any colour at all**, `bar-style.css` with 7 distinct literals and `notifications.conf` with 6, and neither reads the other. No `@define-color`, no shared palette file, no variable crossing a file boundary.

  Four values coincide between them, `#1e1e2e`, `#89b4fa`, `#cdd6f4` and `#f38ba8`, each written out in both places. That is the intended shape and not an oversight: the desktop looks coherent, and the coherence is duplication rather than a mechanism, which is exactly what a later theme change would unify
- [x] 3.4 Named per file, and **one real dependency found rather than none**:

  | File | What a theme would change | Depended on by |
  |---|---|---|
  | `shell/bar-style.css` | every colour literal, the font stack | nothing but the bar |
  | `shell/notifications.conf` | the colour literals and the font; the timeout and anchor are behaviour, not theme | nothing |
  | `shell/wallpaper.conf` | the image path and `fit_mode` | nothing |
  | `compositor/bar.jsonc` | nothing. Replaced by whichever adapter is active, not by a theme | the bar's configuration, by name |
  | `shell/bar.jsonc` | geometry, module placement, formats | nothing |

  **`shell/bar.jsonc` carries one line a wholesale replacement must keep**, the `include` of the adapter's module file. It is composition rather than styling, and a theme change restyles rather than recomposes, so the line the design draws holds. It is recorded here instead of being left for a theme change to discover, which is the whole point of asking the question before there is a theme
- [x] 3.5 All five install through `gridline-config`, read from the built package with `dpkg -c`: four under `/etc/xdg/gridline/shell/` and one under `/etc/xdg/gridline/compositor/`. **Neither of the other two packages ships any path under those directories**, checked rather than assumed, so the package `package-delivery` names as the owner of shipped defaults is the one that owns them
- [x] 3.6 All six files under `/etc` in `gridline-config` are registered, verified by comparing `dpkg-deb -I gridline-config conffiles` against the package's own `/etc` contents entry by entry rather than by eyeballing the list: six shipped, six registered, none missed. Registration is debhelper's default for `/etc` and the check is that nothing escaped it
- [x] 3.7 All three gates pass, and the count is exact. `check-path-ownership` went from **15 files to 20**, which is the five this change ships and nothing more; `check-declared-paths` from 13 to 18 declared paths, 0 not identifiable as Gridline; `check-adapter-isolation` from 4 to 5 inside the adapter and 11 to 15 outside, 0 references found

## 4. Starting them

- [ ] 4.1 Hold all three under `Upholds=` in `gridline-session.target`, not `Wants=`, and verify with `systemctl --user show -p Wants -p Upholds` that the two are reported separately and that the launcher's blocking check, which reads `Wants=`, is unchanged
- [ ] 4.2 Supply whatever the measurements in group 1 showed to be missing: a drop-in where a shipped unit does not take the configuration path this change needs, or a Gridline-authored unit where there is nothing suitable to point at. Verify by reading back the effective `ExecStart` from systemd rather than from the file on disk
- [ ] 4.3 Verify no ordering cycle appears. A target with `DefaultDependencies=yes` gains an implicit `After=` on its `Upholds=` as well as its `Wants=`, measured, and this change takes the target from two entries to five. Check `systemctl --user show -p After` and grep the journal for `ordering cycle`, since the one that happened before was silent except in the journal and produced a reached target with a component never started
- [ ] 4.4 Verify each component is restarted after being killed, which is what `Upholds=` is for and what the scenario about components being kept running asks. Kill each one in turn and confirm it returns
- [ ] 4.5 Verify the session still starts with each component removed in turn, and that the launcher writes nothing refusing it, since none of these is blocking

## 5. Only the Gridline session is affected

**The mechanism this group was written against is gone, and the group is still work.** It assumed Gridline's compositor defaults sat at the compositor's own system configuration path, which every session of that compositor reads, so that an `exec-once` added by this change would start Gridline's components in a session that is not Gridline's. That is no longer true: `confine-gridline-configuration` moved the defaults to a path the session names, and the session-entry diversion stopped the compositor's own entry being offered at the greeter, so the leak is closed by two independent mechanisms rather than one.

A third mechanism removes the premise entirely, and it is the one this change itself chose: **these components are started by the session target, not by an `exec-once`**, so the compositor starts none of them and a session that is not Gridline's reaches no start mechanism at all. The sentence that stood here said the opposite, carried over from the preamble it replaced. The design said both things in two places and now says one.

What that changes is which tasks here are measurements and which are records. It does not make the scenario free: the target is Gridline's, but proving that nothing of this reaches another session is still a measurement against what this change ships rather than one inherited from a change that shipped none of it.

- [x] 5.1 **The failing measurement cannot be obtained on this machine any more, and already exists.** The open finding in `docs/compositor-adapter.md` recorded that every session of that compositor read Gridline's defaults, and `confine-gridline-configuration` measured the closed state against it: the configuration such a session produces is byte for byte the file the machine generated before Gridline was installed, recorded in `docs/configuration.md`. Asking for the failing half again here would mean reinstating the defect in order to watch it, which buys nothing the archived measurement does not already establish
- [ ] 5.2 Verify the route is recorded coherently rather than in halves. Both routes were taken, by `confine-gridline-configuration` and by the session-entry diversion, and three documents now carry part of the answer: `docs/configuration.md`, `docs/package-matrix.md` and `docs/recovery.md`. Read all three and confirm they state the same thing, since the reason this task existed was that two of them each carried half of the question and neither said so
- [ ] 5.3 Verify the scenario passes: start a session other than Gridline on the same machine and confirm none of these components is running in it, reading process state rather than looking at the screen
- [ ] 5.4 Verify the stock Ubuntu session is unchanged, by logging into it, since it is the fallback every recovery path depends on

## 6. Verification in a session

Execution note: this group needs the session running on the VM, started from the greeter. What a user sees is the measurement for several of these and cannot be obtained any other way.

- [ ] 6.1 Verify the machine's state is visible on first login: the current workspace, the time, and the power and network state, each read off the screen by the person using it and each cross-checked against the system's own view so that a bar showing a stale or wrong value is not recorded as a bar working
- [ ] 6.2 Verify a notification sent by an application is shown and then dismissed or expires, using a notification the test sends rather than waiting for one to happen
- [ ] 6.3 Verify a background is displayed, and that it is the one Gridline ships rather than whatever the compositor leaves when nothing has drawn
- [ ] 6.4 Verify a user's own configuration for one of these components replaces Gridline's and is not reimposed, which is what separates a shipped default from policy
- [ ] 6.5 Verify the session still reaches a usable desktop with all three absent, since the requirement says their absence degrades rather than blocks

## 7. Coherence

- [ ] 7.1 Verify each scenario of the added requirement is exercised or recorded as unexercised with a reason, naming the file or measurement for each, and that none is left unaccounted for. Count the scenarios from the delta file rather than from the proposal's description of them
- [ ] 7.2 Verify the capability-gap half of the degradation scenario honestly. Nothing in this change reports a missing component, and the launcher must not look for these or they become blocking; if `add-session-health-check` still does not exist, record the half as owed rather than satisfied
- [ ] 7.3 Verify the elements of the target table in `docs/user-experience.md` that this change delivers are marked as delivered, and that the ones deliberately left out are still readable as out rather than as forgotten

## Workflow follow-up

- Run `/opsx:verify` once the session has been started from the greeter with all three components in place.
- Archive, then the launcher and the command menu, which `docs/user-experience.md` calls the element worth the most to the experience and which this change deliberately left together.
