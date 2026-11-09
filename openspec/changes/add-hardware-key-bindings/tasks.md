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
- [x] 4.3 Verify the keys act while the session is locked, and that the media keys deliberately do not
- [x] 4.4 Verify holding a key repeats it, which the plain bind flavour would not do
- [ ] 4.5 Verify a media key reaches a running player. Record which player was used and that it was addressed through MPRIS rather than by name
- [x] 4.6 Verify the brightness path on a machine with a backlight, or record it as unexercised with the reason and what was verified instead

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

**The keys act while the session is locked, and the lock holds.** With the
session locked by the idle timeout (`LockedHint=yes`, two lockers running) and
the volume at 0.50, one press of volume down gave exactly 0.45. Afterwards the
session was still locked with both lockers still running, so the key adjusted
the volume without unlocking anything. That is the decision the design
defended: silencing a machine reveals and unlocks nothing.

Only volume down was pressed. Mute carries the same flavour and was verified
by its block header, and brightness cannot be tried without a backlight. The
other half of the task, that the media keys deliberately do *not* act while
locked, is established by their headers alone: plain `bind`, not `bindl`. That
is what the compositor's own semantics say, and it was not exercised, because
doing so needs a player running.

## 5. Coherence

- [x] 5.1 Add a row for hardware keys to the experience table in `docs/user-experience.md`, which has none, and state what is delivered versus what waits on the on-screen indicator
- [x] 5.2 Record the on-screen indicator as a known absence where a user would meet it rather than only in this change, since a volume that changes with no feedback is the first thing they will ask about
- [x] 5.3 Verify each scenario of the added requirement is exercised or recorded as unexercised with a reason, counting scenarios from the delta file rather than from the proposal's description of them

## Scenario accounting

Eight scenarios in the added requirement, counted from
`specs/desktop-session/spec.md`. Four were exercised by a person on the keys,
one is exercised in part, and three are recorded as unexercised or partly
exercised with the reason. Nothing below is counted as passing on the strength
of a binding existing.

1. **A volume key changes the volume** - exercised by a person. From a muted
   0.00 the key reached 0.95, nineteen steps of 0.05. One press from 0.20 gave
   exactly 0.25. Read from PipeWire, not from an exit status.
2. **A volume key on a muted output makes it audible** - exercised by a
   person. The same first presses took the output from muted to unmuted. The
   defect itself was found by running the program against a muted output
   (0.30 muted, up gave 0.35 still muted) and fixed before any person pressed
   anything.
3. **Mute is a toggle and is reversible** - exercised in part. A person's one
   press that arrived gave `0.42 [MUTED]` with the level kept. The second
   press never reached the compositor, so the return to 0.42 rests on the
   program run directly, where mute then mute gave 0.42 and 0.42, and on
   `wpctl` preserving the level as measured in group 1.
4. **The keys work while the screen is locked** - exercised by a person for
   volume. Locked by the idle timeout, 0.50 became 0.45 on one press and the
   session stayed locked. Mute shares the flavour and was checked by its
   header only. Brightness could not be tried.
5. **Holding a key repeats it** - exercised by a person. Held from 0.20 it
   reached 1.00 and stopped there.
6. **A media key reaches whatever is playing** - **unexercised.** No player was
   running on the test machine, so no key was seen to reach one. What exists:
   `playerctl` installs from the archive, the program names it, and the three
   bindings are plain `bind` as designed.
7. **Brightness is asked of the system, not taken** - **exercised in pieces.**
   With no backlight the program exits 0 and says so in the journal. logind
   accepts the call's shape, failing with `No such device` and not with a
   signature error. The `session/auto` alias resolves from a child of the
   compositor. And the packages ship no udev rule and change no group, which is
   a statement about their contents. Never exercised against a real backlight.
8. **The keymap names no tool** - exercised. The shipped configuration has no
   occurrence of `wpctl`, `playerctl`, `brightnessctl`, `pamixer` or `busctl`.

**Two tasks stay open and are not closed by this accounting.** Task 4.2 asked
for mute twice to be seen by a person, and only the first press arrived. Task
4.5 asked for a media key to be seen reaching a player, and none was running.
Both can be closed by someone with a few minutes on the test machine, and
neither is closed by writing down why it has not been.

## Workflow follow-up

- Run `/opsx:verify` once the keys have been pressed by a person and the brightness path has been either exercised or recorded as unreachable.
- The on-screen indicator is its own change. `swayosd` ships a udev rule and a system service, which is a desktop shell component's shape and not a keybinding's.
