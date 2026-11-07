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

- [x] 4.1 Held under `Upholds=`, read back from systemd: `Upholds=waybar.service mako.service hyprpaper.service gridline-idle.service`, and `Wants=hyprpolkitagent.service` **alone**, so the launcher's blocking check, which reads `Wants=`, needed no change and still sees one unit
- [x] 4.2 A drop-in per unit, routing `ExecStart` through a Gridline program rather than naming the configuration path in the drop-in itself. Effective `ExecStart` read from systemd, not from disk:

  ```
  waybar     /usr/bin/waybar --config /etc/xdg/gridline/shell/bar.jsonc \
                             --style  /etc/xdg/gridline/shell/bar-style.css
  mako       /usr/bin/mako --config /etc/xdg/gridline/shell/notifications.conf
  hyprpaper  /usr/bin/hyprpaper --config /etc/xdg/gridline/shell/wallpaper.conf
  ```

  **Why the path is not in the drop-in.** A drop-in rewrites `ExecStart` for every session on the machine. Naming the configuration there would make Gridline's defaults reach these components started outside Gridline, which is the leak `confine-gridline-configuration` closed for the compositor. So `/usr/libexec/gridline/gridline-shell <role>` decides at start time against two questions: is this a Gridline session, and would the component find configuration of its own. Outside Gridline it execs the component with no arguments, so the archive's unit behaves as the archive ships it.

  Measured in four cases by reading `/proc/<pid>/cmdline` after the program replaces itself, because the decision is invisible any other way: no desktop identity gives the bare binary; a Gridline session with nothing of the user's gives the binary with Gridline's paths; a planted user configuration gives the bare binary; and an unreadable shipped default gives the bare binary with the reason on stderr, since a missing default degrades rather than blocks.

  **One defect found and fixed, and it would have made the bar's configuration unreachable.** The first version checked `/etc/xdg` among the bar's search bases, as the compositor's check does. The bar's package *ships* `/etc/xdg/waybar/config.jsonc` and `style.css`, so that test read the archive's own default as "the user has configuration of their own" and declined to name Gridline's, every time, on every machine. Measured: `waybar` ran as the bare binary while the other two carried `--config`. The rule is now stated and measured per package: a system base counts only where the component's package ships nothing into it, so that a file being there means somebody put it there. Only the bar ships one. What that gives up is that an administrator cannot override Gridline's bar configuration by editing the archive's file, because nothing can tell that edit apart from the file dpkg installed; the supported route is Gridline's own conffile.

  Also fixed in passing: the launcher imported `XDG_CURRENT_DESKTOP` into the user manager and never unset it, while unsetting `WAYLAND_DISPLAY` and `GRIDLINE_CONFIG` at teardown. The user manager outlives a session, so with lingering enabled the next session of the same user would have read the previous session's desktop identity. It is unset at teardown now, symmetrically with the other two
- [x] 4.6 **The three packages enable their own units at install time, into every graphical session on the machine.** None ships an enable symlink in the `.deb`, which task 1.1 measured correctly and which answers a different question: each postinst runs `deb-systemd-helper --user enable`, and the symlinks are present at `/etc/systemd/user/graphical-session.target.wants/{waybar,mako,hyprpaper}.service`. `graphical-session.target` belongs to every graphical session, so installing Gridline's dependencies starts a bar and a notification daemon in a GNOME session too.

  Task 1.1 saw the same thing happen for `foot` and recorded it as that package's own decision and harmless. For these three it is not harmless: it breaks the delta scenario that no session other than Gridline starts these components, and the additive guarantee that makes GNOME a supported fallback. `gridline-shell` does not help here, because its session check governs which configuration is used and not whether the component runs.

  **Measured in the stock Ubuntu session, and the result is worse than the screen suggests.** Nothing is visible, and both components were started and both failed: the bar five times before systemd gave up, because GNOME implements no layer shell and it cannot create its surface, and the notification daemon once, immediately, with `Failed to acquire service name: File exists` because GNOME already holds it. So a GNOME session on a machine with Gridline installed carries two permanently failed units, and the only reason nothing draws over GNOME is an accident of which protocols the fallback desktop implements rather than anything Gridline arranged.

  **Closed by disabling the three from `gridline-session`'s postinst, restoring them on removal.** The project had already made this exact decision for its own units: `gridline-session.target` says in its own comment that shipping an enable symlink was rejected because `graphical-session.target` belongs to every graphical session. The dependencies undid that behind Gridline's back, and this puts it back.

  It costs Gridline nothing, which is what makes the choice easy rather than a trade. `Upholds=` and `Wants=` start a unit whatever its enabled state, and the machine already demonstrated it: `hyprpolkitagent` reads `disabled` and the session target has started it every session since it was declared.

  `disable` and not `mask`: a masked unit cannot be started by anything, the session target included.

  Measured after installing: the three symlinks are gone from `/etc/systemd/user/graphical-session.target.wants/`, all three read `disabled`, `/var/lib/gridline/disabled-user-units` lists exactly the three it turned off, and the target still upholds all four units. Only units actually turned off are recorded, so removing Gridline does not enable one the administrator had disabled beforehand.

  **A leftover on the VM, not of this change and worth saying.** `foot-server.service` and `foot-server.socket` are still enabled into `graphical-session.target.wants` there, from when `foot` was Gridline's declared terminal. It is no longer a dependency, so the machine carries enable symlinks that current Gridline would not produce, and a later measurement on that machine could read them as current. Cleaning that is the VM's business rather than this change's, and it should happen before group 6 measures what a session starts
- [ ] 4.7 **The wallpaper daemon cannot run on the verification VM**, which blocks the scenario that a background is displayed and leaves a unit crash-looping. It aborts within seconds of being asked to draw, with `KMS: DRM_IOCTL_MODE_CREATE_DUMB failed: Permission denied` first, under systemd and run by hand alike, with a 4K image and with a 70 KB one, so neither the image nor Gridline's configuration is the trigger. That ioctl needs DRM master, which the compositor holds, so the component is choosing a DRM backend rather than the Wayland one; this VM has no KMS-capable device and renders in software.

  `Restart=on-failure` plus `Upholds=` restarts it past its own start limit, so the loop does not settle: `StartLimitBurst=5` in 10s is not reached because `Upholds=` re-upholds a unit that has given up. Measured at **59 starts in two minutes**, roughly one every two seconds, for as long as a session is open, burning CPU and filling the journal the whole time.

  **State to undo on the verification VM:** `hyprpaper.service` is masked there, at `~/.config/systemd/user/hyprpaper.service -> /dev/null`, which is what stops the loop. Unmask it before any measurement that depends on the wallpaper, and remember that a masked unit is not started by the session target either, so its absence from a listing proves nothing while the mask is in place.

  **How far the cause was traced, and where it stopped.** The failing call is `DRM_IOCTL_MODE_CREATE_DUMB` on the primary node, which requires being DRM master. The compositor is master, so any ordinary client is denied: the component is allocating on the primary node rather than the render node, and the library says why in its own words, `DRM dev {} has no render node, falling back to primary`. The device is `virtio-pci` and it is a KMS device, with a connector, so "the VM has no GPU" is not the explanation.

  One real defect was found on the way and was not the cause. The user had no access to `/dev/dri/renderD128` at all: the `uaccess` ACL still named `gdm-greeter` while the user's session was `Active=yes`, so logind had not handed the device over. Re-triggering `uaccess` gave the user the node, and the component failed identically afterwards, which is what rules it out.

  **Not settled: whether this is the component on virtio-gpu or something else about this machine.** It needs a second machine to answer, and guessing from one is how a wrong cause gets recorded as a measured one.

  What is Gridline's regardless of that answer: a non-blocking component that cannot start on a given machine should not spin forever. That is a property of `Upholds=` with `Restart=on-failure`, not of this component, and it is worth settling against observed systemd behaviour rather than against this one crash
- [x] 4.3 **No cycle.** The target's effective `After=` now lists all five units plus `graphical-session.target`, confirming the implicit ordering that `Upholds=` brings with it, and the current user manager logged **zero** ordering cycles against 243 lines logged in total, so the filter is measuring something rather than reading an empty journal.

  The filter is the point. A first reading found four cycles and they were not this change's: `journalctl --user -b` spans the whole boot, and those lines came from two earlier manager instances, `systemd[28701]` and `systemd[32527]`, about `foot-server.socket`, a package since purged. Unit state is per manager instance, and so is its journal; filtering by the running manager's PID is what makes the answer about this session
- [x] 4.4 Killed and returned, with the wrapper running again rather than only the binary: the bar came back as a new PID carrying `--config` and `--style`, and the notification daemon likewise with `--config`. That second half matters, because a restart that lost the configuration path would look identical in a process list that only counts processes.

  The wallpaper daemon satisfies this task and the evidence is worthless, which is worth saying rather than ticking: it restarts constantly without being killed. **22 starts in two minutes**, measured from its journal, and it does not settle. `StartLimitBurst=5` in 10s is reached and `Upholds=` revives it anyway with the counter reset, observed going from `failed` with 5 restarts to `active` with 4 inside a minute. That is a property of asking systemd to keep a component running on a machine where it cannot start, not of this component, and it is the cost of the `Upholds=` half of "kept running"
- [x] 4.5 **The session starts with all three absent, and the launcher says nothing.** Measured from the greeter with the three binaries moved aside: the compositor came up with Gridline's configuration, the session identity was set, the polkit agent and the idle daemon were unaffected and active, the three units failed with `exit-code`, and the launcher's own journal identifier carried no entries at all.

  The evidence that it failed for the reason intended rather than by luck is the wrapper's own message, present once per component and naming both the role and the binary: `gridline-shell: bar: /usr/bin/waybar is not executable`, and the same for `notifications` and `wallpaper`. A unit that failed for any other reason would look identical in a state listing.

  **The task as written is not executable and the substitution is recorded rather than hidden.** It asks for each component removed in turn; all three are `Depends` of `gridline-session`, so apt cannot remove one without removing Gridline itself, and there is no "in turn" that dpkg will perform. Moving the binaries aside exercises the exact branch the wrapper has for an absent component, which is the one this scenario is about. The three were restored afterwards and checked three ways: checksums equal to those taken before the move, `dpkg --verify` clean for all three packages, and modes still `755 root:root`, that last because `dpkg --verify` compares content and would pass a file whose mode had changed.

  The structural half was already settled by reading: the launcher iterates the target's `Wants=`, and these three are in `Upholds=`, so it cannot refuse a session for them. The measurement is what turns that reading into a fact

## 5. Only the Gridline session is affected

**The mechanism this group was written against is gone, and the group is still work.** It assumed Gridline's compositor defaults sat at the compositor's own system configuration path, which every session of that compositor reads, so that an `exec-once` added by this change would start Gridline's components in a session that is not Gridline's. That is no longer true: `confine-gridline-configuration` moved the defaults to a path the session names, and the session-entry diversion stopped the compositor's own entry being offered at the greeter, so the leak is closed by two independent mechanisms rather than one.

A third mechanism removes the premise entirely, and it is the one this change itself chose: **these components are started by the session target, not by an `exec-once`**, so the compositor starts none of them and a session that is not Gridline's reaches no start mechanism at all. The sentence that stood here said the opposite, carried over from the preamble it replaced. The design said both things in two places and now says one.

What that changes is which tasks here are measurements and which are records. It does not make the scenario free: the target is Gridline's, but proving that nothing of this reaches another session is still a measurement against what this change ships rather than one inherited from a change that shipped none of it.

- [x] 5.1 **The failing measurement cannot be obtained on this machine any more, and already exists.** The open finding in `docs/compositor-adapter.md` recorded that every session of that compositor read Gridline's defaults, and `confine-gridline-configuration` measured the closed state against it: the configuration such a session produces is byte for byte the file the machine generated before Gridline was installed, recorded in `docs/configuration.md`. Asking for the failing half again here would mean reinstating the defect in order to watch it, which buys nothing the archived measurement does not already establish
- [x] 5.2 Read all three. **No contradiction, and two real gaps, one of them introduced by this change.**

  The two sentences a grep flags as claiming another session reads Gridline's defaults are both correct: one describes a third-party rebuild's behaviour and the other is in the past tense about the arrangement that was replaced.

  **`docs/package-matrix.md` had a section called "One thing a Gridline package does to another package", and since task 4.6 there are two.** Renamed, split, and the second written out: what it disables, why, measured in the stock Ubuntu session, that it costs nothing because `Upholds=` ignores enabled state, that `disable` is not `mask`, and that this is the one case where a Gridline package writes its own record rather than reading state back, because `dpkg-divert` knows its diversions and nothing knows who disabled a unit. `docs/recovery.md` gains the user-facing half, and each of the two now says it is one of a pair and points at the other: one decides what the login screen offers, the other what starts once a session is running.

  **A defect found while writing the sentence that would have described it.** The draft said Gridline would not undo an administrator's re-enable "because the postinst only acts on a unit it finds enabled", which protects nothing: the postinst runs on every upgrade and on every trigger, so a unit enabled for another session would have been turned off again at the next upgrade, with nothing on the machine explaining why. Fixed in the script rather than in the sentence: a unit named in the record has been dealt with and is never revisited, which also still catches a component installed later than Gridline. Measured by re-enabling the bar, reinstalling `gridline-session`, and reading it back: still `enabled`, record unchanged.

  **`docs/configuration.md` said nothing about the three components**, whose configuration resolves by a mechanism the compositor's section does not describe. It now carries the two questions `gridline-shell` asks, why the path is not in the drop-in, the per-package rule for which search bases count and what it gives up for the bar, and that the wallpaper daemon's shipped format is not the one its documentation shows
- [x] 5.3 **None of the three runs in the stock Ubuntu session**, read from process state and from systemd rather than from the screen: zero processes for each, all three units `inactive` and `disabled`, and Gridline's own `gridline-idle` and the polkit agent inactive too.

  The same session carries **no failed user unit at all**, which is the measurement that makes this mean something. Before task 4.6 disabled the three, the identical check in the identical session found the bar and the notification daemon both failed, so this is a changed state rather than a check that was always going to pass.

  Nothing of Gridline's reaches the session's configuration either: `XDG_CONFIG_DIRS` is `/etc/xdg/xdg-ubuntu:/etc/xdg`, read both from the user manager and from the shell process's own environment, with no Gridline directory on it. `GRIDLINE_CONFIG` is absent, which is the launcher's teardown having removed it
- [x] 5.4 **Unchanged, confirmed by the person using it rather than inferred from unit states.** Logged into the stock Ubuntu session: the dock, the GNOME top bar and the release's own wallpaper are as they were, and nothing reads as different from before Gridline was installed. That last part is the only half of this task a measurement cannot supply, and it is the half the additive guarantee is actually about.

  The notification was the discriminating test and it came out GNOME's: top centre, light rounded panel with a dismiss button, from `org.gnome.Shell.Notifications`, which `busctl` confirms owns the name. The notification daemon Gridline declares would have put a dark box in the top right corner. The terminal in the dock is the baseline's own, not the one Gridline names, which is the desktop-keyed preference file behaving as it should

## 6. Verification in a session

Execution note: this group needs the session running on the VM, started from the greeter. What a user sees is the measurement for several of these and cannot be obtained any other way.

- [x] 6.1 **Three of the four, each cross-checked rather than merely present.** Read off the screen and compared against the system's own view at the same moment: the workspace indicator read `1` and that was the active workspace; the clock read `Fri 06 Nov 17:35` against a system clock reading the same minute; the network module read `wired` against `nmcli` reporting `ethernet:connected` on `enp5s0`. The comparison is the task's point: a bar showing a stale or wrong value looks exactly like a bar working.

  **The power source is not exercised and the reason is the machine, not the bar.** `/sys/class/power_supply/` is empty on the verification VM, so there is no battery for the module to report and `upower` lists only a display device. The module is configured, with three formats separating the power source from the charge, and it has never been seen rendering. Recorded as unexercised rather than counted as passing, and it needs a machine with a battery, which is not something the VM can be made to have
- [x] 6.2 **Shown, and the configuration verified rather than the daemon.** The name `org.freedesktop.Notifications` is held by `mako` running with `--config /etc/xdg/gridline/shell/notifications.conf`, read from the process that owns it. A notification of normal urgency appeared in the top right, in Gridline's dark panel, and expired on its own without being touched.

  The discriminating test is the second one, because a daemon with no configuration at all would have passed the first. A critical notification **stays until dismissed and carries a red border**, which is the `[urgency=critical]` block and nothing else: `default-timeout=0` against the 5000 ms the file sets globally, and `#f38ba8ff` against `#585b70ff`. Two of them stood on screen for over two minutes before `makoctl dismiss --all` cleared them.

  **A reading was wrong before it was right, and the instrument was mine.** The first count of live notifications grepped for a JSON key in output that is not JSON, so `makoctl list` reported zero every time while two notifications were in fact alive, and that zero agreed with a mistaken impression that the critical one had expired. The output it actually prints names each notification and its urgency, and reading it said the opposite
- [ ] 6.3 Verify a background is displayed, and that it is the one Gridline ships rather than whatever the compositor leaves when nothing has drawn
- [x] 6.4 **The user's configuration wins by existing, and the decision is per component rather than global**, which is more than the scenario asks and is the part worth recording.

  A configuration was planted at `~/.config/waybar/config.jsonc`, deliberately unmistakable so that passing could not rest on a subtle detail: the bar at the bottom of the screen, one module, the literal text `USER CONFIG` and a seconds-resolution clock. After a login from the greeter the bar was at the bottom reading `USER CONFIG 17:41:26`, in the component's own default styling rather than Gridline's, with no workspace indicator and no network module. The process line carried neither `--config` nor `--style`: Gridline declined both, which is the rule that naming one while the component finds the other would have broken.

  **In the same session the notification daemon still carried Gridline's path**, because the user had configured the bar and not it. One component deferring does not make the others defer.

  Removing the planted file and restarting the unit brought Gridline's bar back with both paths, so declining is a state rather than a latch, and nothing of the test was left in the user's configuration directory
- [x] 6.5 **Already measured, by task 4.5, and recorded here rather than repeated.** That task logged in from the greeter with all three binaries moved aside: the session reached the desktop, the launcher wrote nothing, the blocking components were unaffected, and each of the three units failed with the wrapper naming the role and the binary it could not run.

  The two tasks ask the same question from the two groups' angles, 4.5 from the start mechanism and this one from what the user sees, and one login cycle answers both. Repeating it would have produced the same evidence at the cost of another cycle, and a verification repeated for the appearance of thoroughness is not a second measurement

## 7. Coherence

- [x] 7.1 **Seven scenarios, counted from the delta file.** Five are exercised, one partly, one not at all, and none is unaccounted for.

  | Scenario | Where | State |
  |---|---|---|
  | The state of the machine is visible | 6.1 | **Partly.** Workspace, clock and network each read off the screen and compared against the system at the same moment. The power source has never been seen rendering: the VM has no battery |
  | A notification reaches the user | 6.2 | **Exercised**, and against the configuration rather than the daemon: a critical notification stays until dismissed with its own border colour |
  | The desktop has a background | 6.3 | **Not exercised.** The daemon aborts before drawing on this machine. 4.7 carries how far the cause was traced |
  | Each component is kept running | 4.4 | **Exercised for two of three.** The bar and the notification daemon were killed and returned, with the wrapper running again rather than only the binary. The wallpaper daemon satisfies it and the evidence is worthless: it restarts without being killed |
  | An absent component degrades and does not block | 4.5, 6.5 | **Exercised** for all three at once, from the greeter, with the wrapper naming each role and binary it could not run |
  | Their configuration is a shipped default a user can decline | 6.4, 4.2 | **Exercised.** Measured in a session for the bar, and by probe for all three, including that one component deferring does not make the others defer |
  | Only the Gridline session is affected | 5.3, 5.4 | **Exercised** in the stock Ubuntu session: none of the three running, no failed unit, no Gridline directory on the configuration search path, and the person who uses it reporting nothing different |

  The requirement this change also modifies carries no task and implements nothing: it shortens prose that stated, in the requirement itself, what one of its own scenarios states word for word. The proposal records that, and no scenario of it is this change's to exercise.

  **One thing this accounting cannot say is that the change is done.** Two of the seven rows are short, and the shortfall is not spread evenly: the background is the whole of one scenario and is the reason 4.7 and 6.3 stay open
- [x] 7.2 **Owed, not satisfied, and the shortfall is larger than this change.** `add-session-health-check` does not exist as a change, active or archived. The launcher's only reporting path is `report_missing`, and every one of its eight call sites ends in `exit 1`: it refuses a session and names the component, which is the blocking case and the opposite of this one. Nothing anywhere reports a component that is absent without blocking.

  Counted rather than estimated: **seventeen cells across five live documents** promise "reported as a capability gap", and two scenarios in `openspec/specs/` require it. Three of those cells are this change's. No document said the promise was unimplemented.

  Recorded in `docs/component-ownership.md`, where eleven of the cells are, stating that what the cells describe about the session is measured and true, and what they promise about being told is owed. An unimplemented promise appearing in seventeen places reads, from any one of them, like a feature.

  **A second count was wrong and the checking found it.** The same section said nine rows carry the degrading fallback. There were eight before this change and there are eleven now, so it was off by one before anyone added anything
- [x] 7.3 **The table had no status at all**, so nothing was marked as anything: it listed each element and the mechanism intended for it, and a reader could not tell the delivered from the planned. It now carries a Status column, read from the shipped files rather than from memory of what was intended, and the three rows that are deliberately out say so with the reason rather than reading as forgotten.

  **It also had no row for notifications**, which this change delivers. Added.

  **Two rows that this change might have been credited with are marked partly, and each names what is missing.** The bar's row asks for workspaces, clock, tray, battery and media; what ships is workspaces, clock, network and battery, with no tray and no media. And the tiling row asks for grouping and a scratchpad among other things, and the shipped compositor defaults contain **zero** bindings for either, counted rather than recalled. The wallpaper reads "shipped, not yet seen", which is the honest state while 4.7 is open

## Workflow follow-up

- Run `/opsx:verify` once the session has been started from the greeter with all three components in place.
- Archive, then the launcher and the command menu, which `docs/user-experience.md` calls the element worth the most to the experience and which this change deliberately left together.
