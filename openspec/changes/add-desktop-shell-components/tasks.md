# Tasks

## 1. The measurements the design deferred

These come first because each one changes how much the session target has to declare. The design names them as open rather than assuming them, after an earlier change got the same question wrong in both directions on two different units.

- [x] 1.1 **None of the three ships an enable symlink**, read from the packages themselves rather than from an installed machine: zero `.wants/` entries in each. All three carry an `[Install] WantedBy=graphical-session.target` that is inert, the same state `hyprpolkitagent` was found in.

  Both unit roots were listed on the machine as the task requires. One entry there is worth recording and is not ours: `foot` enables `foot-server.service` and `.socket` into `/etc/systemd/user/graphical-session.target.wants/`, so a dependency Gridline declared enabled units for every graphical session on the machine, GNOME included. It is that package's own packaging decision and harmless, being socket-activated, but it is a consequence of our dependency rather than of our code
- [x] 1.2 Read from the units, which run a bare binary in all three cases: `/usr/bin/waybar`, `/usr/bin/mako`, `/usr/bin/hyprpaper`. **None passes a configuration path**, so the wallpaper's does not either, which the design left open and assumed would need answering.

  Their shapes differ in a way that matters. `waybar.service` is a plain service. `mako.service` is `Type=dbus` with `BusName=org.freedesktop.Notifications`, and the package ships a matching D-Bus activation file, so it starts on demand when an application sends a notification and needs nothing to want it. `hyprpaper.service` is `Type=simple` with `ConditionEnvironment=WAYLAND_DISPLAY`
- [ ] 1.3 Establish whether the wallpaper daemon honours `--config` as the design assumes, by pointing it at a file outside its search path and confirming a setting from that file takes effect. Verify by observing the running daemon rather than by the absence of an error, since a path it ignores and a path it reads both exit zero
- [x] 1.4 Both can be named explicitly: the binary carries `--config` and `--style`. It also searches `$XDG_CONFIG_HOME/waybar/` then `/etc/xdg/waybar/`, so unlike the other two it has a system path Gridline could ship into.

  Its `include` directive has the precedence **inverted** from the compositor's `source`: the manual states that in case of duplicate options the first defined value wins, including file before included file. So a user who includes Gridline's configuration overrides it by writing their own values anywhere in their file, rather than by writing them after the include
- [x] 1.5 It searches `$XDG_CONFIG_HOME/mako/config` and `~/.mako/config` and **no system path at all**, measured against the binary with a fabricated flag as a negative control. It accepts `--config`, and supports an `include` directive taking an absolute path.

  The question this task asked cannot be answered as posed, because there is no system file for a user file to replace or merge with. What it exposes instead is the reverse problem, recorded in the pause below: passing `--config` is the only way to ship configuration for it, and doing so stops the user's own file being read at all

## 2. Selection and declaration

- [ ] 2.1 Declare the bar, the notification daemon and the wallpaper daemon as `Depends` of `gridline-session`, and verify with `dpkg-deb -f` on the built package rather than by reading the control file
- [ ] 2.2 Verify all three resolve from the archive alone on the baseline VM, recording how many packages apt fetched and the installed size each one brought, since the bar is the largest component the project has declared
- [ ] 2.3 State the `universe` support gap for the three in the `gridline-session` long description, using the wording the live documents already carry, and verify it is identical by comparing against `docs/component-ownership.md`
- [ ] 2.4 Record the three selections in `docs/component-ownership.md` and `docs/package-matrix.md`, with what decided each. For the wallpaper that means recording what the design settles: the rule that rejected two earlier components tests the configuration search and not the library, and a named path is not searched for
- [ ] 2.5 Correct `docs/user-experience.md`, which lists the notification daemon among packages absent from the archive when the archive carries it under a different binary package name. Verify by grep that no live document still records it as unavailable, and leave the other four names on that line alone, since they were checked and are genuinely absent

## 3. Configuration and styling

**Paused here, and the pattern comes from elsewhere.** Group 1 found that only the bar has a system configuration path, and then that using it would repeat a violation already shipped: `gridline-config` declares `/etc/xdg/hypr/hyprland.conf`, and `package-delivery` requires every declared path to carry a Gridline-identifying name. No gate caught it, because the ownership gate tests for collisions between Gridline packages rather than for Gridline-named-ness, and the manifest line that declares it sits in `debian/`, outside the isolation check.

The answer for all three is the same as the answer for the compositor: a Gridline-named path, named explicitly with `--config`, rather than a file placed where the component searches. `confine-gridline-configuration` establishes that for the compositor and adds the gate; this change inherits the pattern rather than inventing it, which is why these tasks wait.



- [ ] 3.1 Author each component's own configuration, in that component's format, outside the adapter path, since these are read by programs that are not the compositor. Verify the isolation check still reports zero references outside the adapter path, and that it does so with the name match as well as the content match
- [ ] 3.2 Author the compositor-side configuration that launches and places these components, inside the adapter path, since that part is the compositor's vocabulary
- [ ] 3.3 Ship a default appearance for each, and verify against the line the design draws: no colour is defined once and consumed by two components. Confirm by reading the shipped files, since the test is about whether anything generalises rather than about how much styling there is
- [ ] 3.4 Verify a later theme change could replace these files wholesale without unpicking anything, by naming for each file what would have to change and confirming none of them is a mechanism other files depend on
- [ ] 3.5 Ship all of it through `gridline-config` and verify with `dpkg -c` on the built package that the installed paths belong to the package `package-delivery` names as the owner of shipped defaults
- [ ] 3.6 Register the shipped configuration as conffiles and verify with `dpkg-deb -I <package> conffiles`, since an administrator is expected to edit them
- [ ] 3.7 Verify both gates pass and account for the ownership gate's file count, which must rise by exactly the number of files this change ships

## 4. Starting them

- [ ] 4.1 Hold all three under `Upholds=` in `gridline-session.target`, not `Wants=`, and verify with `systemctl --user show -p Wants -p Upholds` that the two are reported separately and that the launcher's blocking check, which reads `Wants=`, is unchanged
- [ ] 4.2 Supply whatever the measurements in group 1 showed to be missing: a drop-in where a shipped unit does not take the configuration path this change needs, or a Gridline-authored unit where there is nothing suitable to point at. Verify by reading back the effective `ExecStart` from systemd rather than from the file on disk
- [ ] 4.3 Verify no ordering cycle appears. A target with `DefaultDependencies=yes` gains an implicit `After=` on its `Upholds=` as well as its `Wants=`, measured, and this change takes the target from two entries to five. Check `systemctl --user show -p After` and grep the journal for `ordering cycle`, since the one that happened before was silent except in the journal and produced a reached target with a component never started
- [ ] 4.4 Verify each component is restarted after being killed, which is what `Upholds=` is for and what the scenario about components being kept running asks. Kill each one in turn and confirm it returns
- [ ] 4.5 Verify the session still starts with each component removed in turn, and that the launcher writes nothing refusing it, since none of these is blocking

## 5. Only the Gridline session is affected

The delta makes this a scenario rather than a note, so it is work rather than a caveat. Gridline's compositor defaults sit at the compositor's own system configuration path, which every session of that compositor reads, so an `exec-once` added by this change starts Gridline's components in a session that is not Gridline's.

- [ ] 5.1 Verify the scenario fails before the change, so that passing it afterwards means something: with the current defaults in place, confirm a plain compositor session inherits Gridline's configuration
- [ ] 5.2 Close it, by whichever of the two routes the project takes: the compositor configuration reaching only the Gridline session, or the other session entry no longer being offered. Record which route and why, since `docs/compositor-adapter.md` and `docs/package-matrix.md` each already carry half of this question
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
