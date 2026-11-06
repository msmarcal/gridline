# Tasks

## 1. The measurements the design deferred

These come first because each one changes how much the session target has to declare. The design names them as open rather than assuming them, after an earlier change got the same question wrong in both directions on two different units.

- [x] 1.1 **None of the three ships an enable symlink**, read from the packages themselves rather than from an installed machine: zero `.wants/` entries in each. All three carry an `[Install] WantedBy=graphical-session.target` that is inert, the same state `hyprpolkitagent` was found in.

  Both unit roots were listed on the machine as the task requires. One entry there is worth recording and is not ours: `foot` enables `foot-server.service` and `.socket` into `/etc/systemd/user/graphical-session.target.wants/`, so a dependency Gridline declared enabled units for every graphical session on the machine, GNOME included. It is that package's own packaging decision and harmless, being socket-activated, but it is a consequence of our dependency rather than of our code
- [x] 1.2 Read from the units, which run a bare binary in all three cases: `/usr/bin/waybar`, `/usr/bin/mako`, `/usr/bin/hyprpaper`. **None passes a configuration path**, so the wallpaper's does not either, which the design left open and assumed would need answering.

  Their shapes differ in a way that matters. `waybar.service` is a plain service. `mako.service` is `Type=dbus` with `BusName=org.freedesktop.Notifications`, and the package ships a matching D-Bus activation file, so it starts on demand when an application sends a notification and needs nothing to want it. `hyprpaper.service` is `Type=simple` with `ConditionEnvironment=WAYLAND_DISPLAY`
- [x] 1.3 **It honours `--config`, and the design's assumption holds.** Measured on the VM against the running compositor, with `strace -e trace=openat` on the daemon rather than its exit code: with the flag it opens the named file outside every path it searches, `openat(AT_FDCWD, "/tmp/gridline-probe/good.conf", O_RDONLY) = 3`; without it, and with nothing in the search path, it opens no configuration file at all. The two cases differ in exactly that one syscall, which is the discriminating pair.

  Three instruments were wrong before that one, and the third found the answer only because the first two had controls. Run headless with no Wayland display, the daemon dies in `XDG_RUNTIME_DIR` with SIGSEGV **before reading any configuration**, and a nonexistent config path produces output identical to a valid one, so the whole first attempt measured nothing. `hyprctl hyprpaper listloaded` answers `Unknown hyprpaper request`: 0.8.x moved its IPC to a Wayland protocol, so the documented observation point no longer exists. And `--verbose` logs nothing about configuration whatsoever: four cases, including an invalid directive, produced logs of 90 lines that are byte-identical once addresses are masked

- [ ] 1.6 **A finding for group 3, not for this group: the configuration format changed and the published syntax does not apply.** The named file is read and nothing in it takes effect. The reason is not the flag: `preload`, `wallpaper`, `splash` and `ipc`, the directives every current example uses, **do not exist as strings in the 0.8.3 binary**, measured for each one. What is there is `hyprpaper_wallpaper`, a `Failed parsing wallpaper for key` message, `splash_offset`, `source=`, and Hyprlang's `addSpecialCategory`, so the shape is a special category rather than the flat keys of 0.7. The changelog agrees this is a rewrite: 0.8.0 dropped cairo, pango and hyprgraphics for hyprtoolkit. Establish the real syntax from the 0.8.3 source or its own documentation before task 3.1 authors a file, and do not take it from an example written against 0.7, which parses without complaint and does nothing
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

**The pattern these tasks waited for is settled, and they are no longer paused.** Group 1 found that only the bar has a system configuration path, and then that using it would repeat a violation already shipped: `gridline-config` declared `/etc/xdg/hypr/hyprland.conf`, and `package-delivery` requires every declared path to carry a Gridline-identifying name. No gate caught it, because the ownership gate tests for collisions between Gridline packages rather than for Gridline-named-ness, and the manifest line that declared it sat in `debian/`, outside the isolation check.

`confine-gridline-configuration` closed that and these tasks inherit its result rather than inventing one. The answer for all three is the answer for the compositor: a Gridline-named path, named explicitly when the session starts the component rather than placed where the component searches, and named **only when the component's own search would find nothing**, so that a user's own configuration wins by existing. That last clause is the part worth carrying over, because naming a path unconditionally takes the user's configuration away from them, which is the problem the compositor had to solve and which `mako` has in a sharper form: group 1 measured that it searches no system path at all, so `--config` is the only way to ship configuration for it and passing it stops the user's file being read.

The gate that was missing exists: `tools/check-declared-paths` fails a manifest line whose path carries no project name, and it matches on any segment rather than the whole path, so a Gridline-named file in a directory another component owns still passes.



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

**The mechanism this group was written against is gone, and the group is still work.** It assumed Gridline's compositor defaults sat at the compositor's own system configuration path, which every session of that compositor reads, so that an `exec-once` added by this change would start Gridline's components in a session that is not Gridline's. That is no longer true: `confine-gridline-configuration` moved the defaults to a path the session names, and the session-entry diversion stopped the compositor's own entry being offered at the greeter, so the leak is closed by two independent mechanisms rather than one.

What that changes is which tasks here are measurements and which are records. It does not make the scenario free: an `exec-once` is still how these three components start, and this change adds three of them, so the scenario has to be measured against what this change ships rather than inherited from a change that shipped nothing of it.

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
