# Tasks

## 1. What the keys do before anything is bound

Three components before this one were measured before being configured, and each time the measurement changed what got built. Here the pattern already paid once: brightness looked like a group-membership problem until logind was introspected.

- [x] 1.1 Establish what each of the nine keys does today in a Gridline session: press each and record whether anything happens. The expected answer is nothing, and a key that already does something is a key this change must not take over
- [x] 1.2 Establish how the compositor reports a binding that uses the repeat and locked flavours, by adding one and reading it back from `hyprctl binds` rather than from the file. The flavours are the part of this change the shipped configuration has never used
- [x] 1.3 Establish the exact `wpctl` behaviour this will depend on: what `set-volume` does at the ceiling, whether it clamps or exceeds 100%, and what `set-mute toggle` reports. Measure rather than assume, because the requirement says mute restores the user's own level

### What group 1 measured

**None of the nine is bound, and nothing else wants them.** The compositor
knows 42 bindings and not one names a hardware key. An earlier reading
suggested two `gnome-settings-daemon` processes were running and might be
grabbing media keys; that was `pgrep -f` matching the shell that ran it, and
no GNOME component runs in the session at all.

**The flavours are reported as the block header, not as a field.**
`hyprctl binds` prints `modmask, submap, key, keycode, catchall, description,
dispatcher, arg` and nothing about repeat or locked. What carries it is the
keyword heading each block: `bind`, `binde`, `bindl`. A check looking for a
field finds nothing and concludes the flavour was lost.

**And the combined form is normalised.** Writing `bindel` reads back as
`bindle`. A verification comparing the header against what was written would
fail on the one binding that uses both flavours, which is exactly the one
worth getting right.

**`wpctl` exceeds 100% unless told not to.** From a volume of 1.0, `5%+`
gives 1.05. With `-l 1.0` it clamps to 1.00. So the limit is part of the
implementation rather than a refinement.

**And `set-mute toggle` preserves the level by itself.** Measured: 0.42, muted
at 0.42, unmuted at 0.42. The requirement that mute restores the user's own
level is satisfied by the tool, and the session needs to remember nothing.

## 2. The program the keys name

- [x] 2.1 Add the Gridline program with a role per action, following the shape `gridline-menu` and `gridline-shell` already use. Verify it rejects an unknown role and names no tool in the keymap, by reading both the program and `defaults.conf` back
- [x] 2.2 Implement volume up, down and mute through `wpctl`. Verify each against the system's own view of the volume rather than against the command's exit status
- [x] 2.3 Implement microphone mute. Verify against the default source's reported state, and record which device the default resolves to on this machine
- [x] 2.4 Implement brightness by reading `/sys/class/backlight/*/brightness` and `max_brightness` and calling `SetBrightness` on logind. Verify the call is accepted; on a machine with no backlight, verify the program says so once and exits zero rather than failing
- [x] 2.5 Implement the media keys through `playerctl`, and declare it in `debian/control`. Verify the dependency is satisfiable from the archive by its origin, not by the install already having it

### What group 2 measured

**Volume, mute and the ceiling behave as group 1 predicted.** Read from
PipeWire, not from exit statuses: up and down move by 0.05; from 1.0 the up key
stays at 1.00 where bare `wpctl` went to 1.05; mute twice returns 0.42 to 0.42.

**This machine cannot tell the microphone from the speakers.** The default
sink and the default source both resolve to `auto_null`, the dummy node
PipeWire creates when there is no audio hardware. Muting the microphone mutes
the sink, and setting the source's volume sets the sink's. The mic-mute key was
verified to act on the node `@DEFAULT_AUDIO_SOURCE@` names, and nothing more:
a bug that pointed it at the sink would have passed identically. Closing that
needs a machine with real input and output devices.

**The brightness path was verified in pieces, because the whole is out of
reach.** With no backlight the program writes one line to stderr and exits 0.
The call shape is accepted by logind: a nonexistent device fails with `Failed
to open device backlight:nao-existe: No such device`, which is an error about
the device and not about the signature. And the `session/auto` alias, which
fails from a process in no session, resolves from a child of the compositor,
tested with `hyprctl dispatch exec` because that is the context the key press
runs in.

**The design said the program would say so once and the first code said
nothing.** Fixed in the code rather than the design: a stderr line, which is
the journal and not the screen.

**The bindings loaded as designed.** Read from the block header, since that is
where the compositor puts the flavour: the four volume and brightness keys as
`bindle` (the compositor's name for `bindel`), mute and microphone mute as
`bindl`, the media keys as plain `bind`. Fifty-one bindings, none colliding.
`playerctl` comes from `archive.ubuntu.com resolute/universe`, and the keymap
names no tool.

## 3. The bindings themselves

- [x] 3.1 Add the nine bindings to `defaults.conf` with the repeat flavour on volume and brightness and the locked flavour on volume, brightness and mute, and without it on the media keys. Verify from `hyprctl binds` that each carries the flavour intended, which is the thing a typo makes invisible
- [x] 3.2 Verify every new binding is a key the file did not already bind, by listing all bindings before and after and comparing. The file binds `L` twice over for movement and focus already, so a collision here would silently take one of the two away
- [x] 3.3 Verify the build gates still pass, including the one that keeps compositor vocabulary inside the adapter

## 4. Measured on a running session

- [x] 4.1 Verify a volume key changes the volume, pressed by a person, and that the system's own view reflects it
- [ ] 4.2 Verify mute twice returns the volume to the level it had rather than to one the session chose
- [ ] 4.3 Verify the keys act while the session is locked, and that the media keys deliberately do not
- [x] 4.4 Verify holding a key repeats it, which the plain bind flavour would not do
- [ ] 4.5 Verify a media key reaches a running player. Record which player was used and that it was addressed through MPRIS rather than by name
- [ ] 4.6 Verify the brightness path on a machine with a backlight, or record it as unexercised with the reason and what was verified instead

### What a person pressing the keys established

Run on the project's VM with a person at the keyboard, reading the audio
state from PipeWire after each step rather than taking a description of what
was heard.

**Volume up works end to end.** From a muted 0.00 the key unmuted the output
and raised it to 0.95, nineteen steps of 0.05, which only this program does.
A single press from 0.20 gave exactly 0.25. Holding from 0.20 reached 1.00 and
stopped there, so repeat works and the ceiling held with a person on the key
where bare `wpctl` would have gone past it.

**Mute worked once in four attempts, and the cause of the other three is
unknown.** One press produced `0.42 [MUTED]` and exactly one line in a
temporary log bound to the same key, at 00:14:05: the compositor saw it once,
the program muted the sink, and the level survived. The other three presses
left neither the volume nor either of two logs changed. A second logging bind
on `F1` to `F12` was added to catch a keyboard sending function keys instead,
and caught nothing. So those presses never reached the compositor. The likeliest
reasons are the host desktop consuming the key or the VM window not holding
focus, and neither can be told apart from the VM. The same host already
swallows `Ctrl+Alt+F3` for the VM.

**So pressing mute twice to restore the level was not completed by a person.**
Its first half was: mute, observed, level preserved. Its second half rests on
the direct runs of the program, where mute then mute returned 0.42 to 0.42,
and on `wpctl set-mute toggle` preserving the level as measured in group 1.

## 5. Coherence

- [x] 5.1 Add a row for hardware keys to the experience table in `docs/user-experience.md`, which has none, and state what is delivered versus what waits on the on-screen indicator
- [x] 5.2 Record the on-screen indicator as a known absence where a user would meet it rather than only in this change, since a volume that changes with no feedback is the first thing they will ask about
- [ ] 5.3 Verify each scenario of the added requirement is exercised or recorded as unexercised with a reason, counting scenarios from the delta file rather than from the proposal's description of them

## Workflow follow-up

- Run `/opsx:verify` once the keys have been pressed by a person and the brightness path has been either exercised or recorded as unreachable.
- The on-screen indicator is its own change. `swayosd` ships a udev rule and a system service, which is a desktop shell component's shape and not a keybinding's.
