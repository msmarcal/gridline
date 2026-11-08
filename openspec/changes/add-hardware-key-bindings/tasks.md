# Tasks

## 1. What the keys do before anything is bound

Three components before this one were measured before being configured, and each time the measurement changed what got built. Here the pattern already paid once: brightness looked like a group-membership problem until logind was introspected.

- [ ] 1.1 Establish what each of the nine keys does today in a Gridline session: press each and record whether anything happens. The expected answer is nothing, and a key that already does something is a key this change must not take over
- [ ] 1.2 Establish how the compositor reports a binding that uses the repeat and locked flavours, by adding one and reading it back from `hyprctl binds` rather than from the file. The flavours are the part of this change the shipped configuration has never used
- [ ] 1.3 Establish the exact `wpctl` behaviour this will depend on: what `set-volume` does at the ceiling, whether it clamps or exceeds 100%, and what `set-mute toggle` reports. Measure rather than assume, because the requirement says mute restores the user's own level

## 2. The program the keys name

- [ ] 2.1 Add the Gridline program with a role per action, following the shape `gridline-menu` and `gridline-shell` already use. Verify it rejects an unknown role and names no tool in the keymap, by reading both the program and `defaults.conf` back
- [ ] 2.2 Implement volume up, down and mute through `wpctl`. Verify each against the system's own view of the volume rather than against the command's exit status
- [ ] 2.3 Implement microphone mute. Verify against the default source's reported state, and record which device the default resolves to on this machine
- [ ] 2.4 Implement brightness by reading `/sys/class/backlight/*/brightness` and `max_brightness` and calling `SetBrightness` on logind. Verify the call is accepted; on a machine with no backlight, verify the program says so once and exits zero rather than failing
- [ ] 2.5 Implement the media keys through `playerctl`, and declare it in `debian/control`. Verify the dependency is satisfiable from the archive by its origin, not by the install already having it

## 3. The bindings themselves

- [ ] 3.1 Add the nine bindings to `defaults.conf` with the repeat flavour on volume and brightness and the locked flavour on volume, brightness and mute, and without it on the media keys. Verify from `hyprctl binds` that each carries the flavour intended, which is the thing a typo makes invisible
- [ ] 3.2 Verify every new binding is a key the file did not already bind, by listing all bindings before and after and comparing. The file binds `L` twice over for movement and focus already, so a collision here would silently take one of the two away
- [ ] 3.3 Verify the build gates still pass, including the one that keeps compositor vocabulary inside the adapter

## 4. Measured on a running session

- [ ] 4.1 Verify a volume key changes the volume, pressed by a person, and that the system's own view reflects it
- [ ] 4.2 Verify mute twice returns the volume to the level it had rather than to one the session chose
- [ ] 4.3 Verify the keys act while the session is locked, and that the media keys deliberately do not
- [ ] 4.4 Verify holding a key repeats it, which the plain bind flavour would not do
- [ ] 4.5 Verify a media key reaches a running player. Record which player was used and that it was addressed through MPRIS rather than by name
- [ ] 4.6 Verify the brightness path on a machine with a backlight, or record it as unexercised with the reason and what was verified instead

## 5. Coherence

- [ ] 5.1 Add a row for hardware keys to the experience table in `docs/user-experience.md`, which has none, and state what is delivered versus what waits on the on-screen indicator
- [ ] 5.2 Record the on-screen indicator as a known absence where a user would meet it rather than only in this change, since a volume that changes with no feedback is the first thing they will ask about
- [ ] 5.3 Verify each scenario of the added requirement is exercised or recorded as unexercised with a reason, counting scenarios from the delta file rather than from the proposal's description of them

## Workflow follow-up

- Run `/opsx:verify` once the keys have been pressed by a person and the brightness path has been either exercised or recorded as unreachable.
- The on-screen indicator is its own change. `swayosd` ships a udev rule and a system service, which is a desktop shell component's shape and not a keybinding's.
