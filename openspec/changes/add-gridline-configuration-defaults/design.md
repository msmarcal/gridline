# Design

## Context

See proposal.md for motivation. Four measurements and one apparent rule conflict shape the approach.

**The compositor resolves configuration by first file wins.** `docs/configuration.md` records the order `Hyprutils::Path::findConfig` checks: `$XDG_CONFIG_HOME`, then `$HOME/.config`, then each entry of `XDG_CONFIG_DIRS`, then `/etc/xdg`, returning the first file it finds and stopping. A shipped default is therefore read when the user has none, and is never read at all once the user writes one. There is no merge and no per-setting override.

**`source` is a real directive.** The compositor binary carries the line `# source = ~/.config/hypr/myColors.conf` in the template it writes when no configuration exists, so a configuration file can pull in another. Whether it accepts an absolute path is not established and is the first thing the implementation measures, because the whole approach below rests on it.

**`SetLockedHint` is a method on the logind session object.** Introspection of `/org/freedesktop/login1/session/_317` lists `.SetLockedHint` taking a boolean, `.LockedHint` as a property, and `.Lock` as both a method and a signal. So the locked state is something a session tells logind; nothing infers it.

**The selected locker and the candidate idle daemon do not set it.** `LockedHint` read `no` through both lock verifications in `add-session-lifecycle-integration`, with the compositor genuinely locked. `swayidle`'s manual documents `idlehint`, which sets `IdleHint`, and no equivalent for the locked state.

**Two project rules appear to point at different places for one file.** `package-delivery` requires `gridline-config` to own the shipped defaults under `/etc/xdg`. `compositor-abstraction`, enforced by `tools/check-adapter-isolation`, requires compositor-specific content to live inside the adapter path. A compositor configuration file is both the most compositor-specific content in the project and a shipped default.

## Goals / Non-Goals

**Goals:**

- A first login that is operable from the keyboard, with Gridline's defaults in effect.
- A customization path that does not require the user to discard every default to change one thing.
- Locking that happens without being asked, and a locked state the rest of the system agrees with.
- A resolution of the rule conflict that holds for the next compositor as well as this one.

**Non-Goals:**

- Enforceable policy. The generator, per-setting locks and the reporting of unapplied policy are `session-manageability`'s, and shipping a default an administrator may edit is deliberately not that.
- The rest of the experience table: bar, launcher, wallpaper, capture tools, terminal and editor defaults.
- Theming. A theme is data `gridline-config` will own, and nothing here chooses one.

## Decisions

### The rule conflict is not a conflict, because the two rules are about different axes

`package-delivery` says which **binary package** owns an installed path. `compositor-abstraction` says where in the **source tree** compositor-specific content may live. Reading both as statements about the same axis is what made them look contradictory.

So the compositor's configuration is authored under `src/adapters/<compositor>/`, inside the isolation boundary where compositor-specific content belongs, and is installed into `/etc/xdg` by `gridline-config`, which owns shipped defaults. The `.install` line that connects them sits in `debian/`, which the isolation check already excludes for exactly this reason: packaging metadata has to name the things the project depends on.

The test this has to pass is the one `compositor-abstraction` cares about: supporting a second compositor is adding an adapter, not editing a file that knows a compositor's name. Under this arrangement a second compositor brings its own configuration inside its own adapter directory, and nothing outside changes.

### The gate matches contents and should also match names

`tools/check-adapter-isolation` greps file contents for eleven reference strings. A compositor configuration file placed outside the adapter path would pass it: `bind = SUPER, Q, killactive` contains none of the eleven, while being entirely compositor-specific. The gate would have been silent about the exact mistake this change could most easily make.

The fix is small and keeps the gate's character, which is that the path decides rather than a reviewer: match the reference list against file **names** as well as contents, so a file called `hyprland.conf` outside the adapter path fails whatever is in it. This is verified the way the list already is, by planting a file and confirming the build fails.

It does not make the gate complete. A compositor configuration under a neutral name still passes, and that is a limit to record rather than to chase: a content check cannot recognise a syntax it does not know.

### Defaults are shipped as a thin loader plus a Gridline-owned body

Shipping the whole default set at `/etc/xdg/hypr/hyprland.conf` works until the user writes their own file, at which point every default disappears at once. That is the behaviour `docs/configuration.md` identifies as the obstacle, and building the first-login experience directly on it would hand the user a choice between Gridline's defaults and any customization at all.

Instead the defaults live in a Gridline-named file, and what sits at the compositor's search path is a small file that sources it. A user who wants to customize copies that loader into their own configuration directory and edits around the `source` line, keeping the defaults and adding to them. The mechanism is the component's own, not something Gridline invents.

Two things this does not do, stated because the obvious reading overclaims. It does not make the defaults unoverridable: a user who deletes the `source` line loses them, which is correct, because enforcement is `session-manageability`'s and a default is not policy. And it does not merge per setting; the user's additions come after the sourced file and win by order, which is the compositor's rule rather than a Gridline guarantee.

**If `source` does not accept an absolute path**, this approach needs the loader to reference a path under the user's own directory, and the implementation reports that before building on it rather than discovering it at verification. That is why the measurement is the first task.

### Which configuration is in effect has to be answerable

The third scenario of the new requirement asks that a user not lose the defaults silently. Silence is the actual failure mode: a session with no Gridline defaults starts, runs, and looks like a session with them until the user presses a key that does nothing.

So the defaults carry something a running session can be asked about, and the session can report which configuration produced its current state. What that marker is depends on what the compositor can be asked at runtime, which is a measurement rather than a choice, and the adapter is where the asking belongs, since it is the component that already queries the compositor. `add-session-health-check` owns surfacing the answer; this change owns making the answer exist.

### A terminal is declared, and the binding does not name it

The keymap scenario asks that a user be able to launch a terminal on first login. The baseline does not supply one: measured against the archive, neither `ubuntu-desktop` nor `ubuntu-desktop-minimal` depends on or recommends a terminal emulator, and both depend on `xdg-terminal-exec`. The only terminal on the verification machine is one that `apt-mark showmanual` reports as hand-installed, so it is not evidence about the baseline. Every candidate is in `universe`, so that gap applies whichever is chosen and the choice rests elsewhere.

The binding calls `xdg-terminal-exec`, which is in `main` and is the freedesktop mechanism for "open the terminal this system is configured to use". `gridline-session` separately declares `foot`, so there is one for it to find. Two things follow, and both are the point rather than a side effect: a user who prefers another terminal registers it the standard way instead of editing a Gridline keybinding, and Gridline's keymap contains no terminal's name, so the choice can change without the configuration changing.

`foot` over the alternatives for the ordinary reasons: it is the terminal `docs/user-experience.md` already names in the 3.x stack this project targets, it is Wayland-native, and at 700 KB installed it is the smallest of the candidates against `gnome-console`'s 1164 KB. Configuring it stays out of scope; this change declares that it exists and nothing more.

### `swayidle`, on the evidence already gathered

`add-session-lifecycle-integration` measured the candidates and recorded them as candidate evidence rather than discarding them: `swayidle` links no Hyprland library, where `hypridle` links `libhyprlang2` and `libhyprutils10`. The reason that mattered there applies here with more force, because this is the change that puts the idle daemon in the lock path: `libhyprlang`'s `findConfig` is the obstacle this whole design works around, and building the component that decides when to lock on top of it would be a choice rather than an inheritance.

Its manual documents the events this needs: `timeout` for inactivity, `lock` for the logind `Lock` signal, `before-sleep` for suspend, and `idlehint` for the idle state. The one thing it does not document is anything that sets the locked state.

### One lock path, entered from three directions

Manual locking, idle locking and locking before suspend must not be three implementations that can disagree, because `session-manageability` already requires that lock policy be one policy rather than several components configured independently. This change does not build the policy layer, but it should not build something the policy layer will have to undo.

So there is one Gridline-owned lock entry point, and everything routes through it: the idle timeout, the suspend hook, and a key binding that asks logind to lock rather than running the locker directly. Asking logind means the `Lock` signal reaches the idle daemon's `lock` event, so a lock requested from anywhere on the system arrives by the same route.

That entry point is also what sets `LockedHint`: true before the locker starts, false after it exits. It is Gridline-authored glue over archive tools, which `docs/user-experience.md` already classifies as Gridline's own work rather than a setting.

The failure to be careful about is the one that leaves the session exposed. If the hint is set and the locker then fails to start, the system believes a session is locked that is not. So the order matters and the implementation verifies it: the hint is cleared when the locker exits, whatever the reason it exits.

## Risks / Trade-offs

- **The loader approach depends on a directive whose behaviour with absolute paths is unmeasured.** → Measured first, before anything is built on it, and the design states what changes if it does not hold.
- **A user who deletes the `source` line loses the defaults.** → Correct by design, and the new requirement asks only that it not be silent. Making it impossible is policy, which is out of scope and specified elsewhere.
- **Gridline now authors a lock entry point, which is code in the fail-safe path.** → It is a wrapper around the archive's locker rather than a locker, and the risk it adds is a wrongly set hint rather than a failure to lock. The scenario that already exists for lock failure covers the component underneath.
- **The gate improvement will not catch a compositor configuration under a neutral name.** → Recorded as a limit. The gate's rule is that the path decides, so the answer is the path arrangement above, not a cleverer grep.
- **Shipping a `misc:allow_session_lock_restore` default changes compositor behaviour on a security-relevant path.** → It is what makes the measured lock-failure state recoverable from the machine itself, and the new scenario asks for exactly that. The setting permits restoring a lock screen, not bypassing one.

## Migration Plan

Installing this version gives `gridline-config` content for the first time, so a machine that already has Gridline gains defaults it did not have. The shipped defaults are conffiles, which is what keeps an administrator's edits across upgrade, and the generated configuration the policy layer will later produce is not a conffile for the reason `docs/configuration.md` records.

A user who has already written their own compositor configuration keeps it and sees no change, which is the first-file-wins rule working as documented rather than an upgrade failure. They adopt the defaults by adding a `source` line.

Rollback is the previous package version. Nothing outside Gridline's own paths is modified, so removing the package returns the compositor to its autogenerated default.
