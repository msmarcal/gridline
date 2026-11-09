# Tasks

## 1. Before anything is built

The design rests on claims taken from one log on one laptop and on how this compositor's output keyword behaves. Each is checked on the VM where it can be, and what cannot be is named, so that the group that is left for the owner's laptop is a short list and not a surprise.

- [x] 1.1 Create a headless output on the VM's running compositor to stand in for an external screen, disable and re-enable it with the compositor's own output keyword, and read the monitor list after each. Record the keyword forms that work, including the one that returns an output to its preferred mode, position and scale
- [x] 1.2 With two outputs and a window on the one that is then disabled, read where the window and its workspace are afterwards. The design expects the compositor to move them; this reads it
- [x] 1.3 Disable an output, then make the compositor reload its configuration, and read whether the output is back. The design expects it not to survive; this reads it
- [x] 1.4 Confirm on the VM that the connector names in `/sys/class/drm` and the compositor's output names are the same strings, and read what the VM's one output is called, so the naming claim is checked somewhere other than the laptop it came from
- [x] 1.5 Load the two lid-switch bindings into the running compositor and read its configuration errors and its binding list, so a syntax the compositor refuses is found before it is shipped

Findings so far:

- **1.4** On the VM, `/sys/class/drm` has `card1-Virtual-1` (connected) and the bar, which reads output names from the compositor, logged `Virtual-1` in all 22 sessions it ran. On the laptop the kernel connector is `card1-eDP-1` and the compositor's own log says `Connector gets name eDP-1`. The same strings in both places on both machines.
- **Attempted first, and not possible without a login:** a headless compositor with no session. `AQ_BACKENDS=headless` is not read by this aquamarine (its only `AQ_` variables are for DRM) and `HYPRLAND_HEADLESS_ONLY` is not in the binary; both runs ended in `CBackend::create() failed!`, the failure the hosted runner showed, and left two compositor processes running that were stopped by hand. The rest were run from a logged-in Gridline session on the VM, which can add outputs with `hyprctl output create headless`.
- **1.1** Two headless outputs were created (`HEADLESS-2`, `HEADLESS-3`) beside the real `Virtual-1`. `hyprctl keyword monitor "HEADLESS-3, disable"` printed `ok`, and the active monitor count went from 3 to 2 while the output stayed in the list of all monitors. `hyprctl keyword monitor "HEADLESS-3, preferred, auto, auto"` printed `ok`, the count returned to 3, and the output returned at its mode and position (`1920x1080` at `2240x0`). Those two forms are the verb's implementation.
- **1.2** A window opened on `HEADLESS-3` (workspace 3, monitor 2). After disabling that output the window still existed and its workspace was on monitor 0, `Virtual-1`; after enabling it, the workspace was back on `HEADLESS-3`. The compositor moves a disabled output's workspaces to a remaining output and returns them, so the windows are reachable.
- **1.3** A disabled output does not survive a reload: 2 active before `hyprctl reload`, 3 after. This is what the design expected, and it is the accepted limit.
- **1.5** Loading the two bindings with `hyprctl keyword bindl` printed `ok` twice and left `configerrors` empty. The switch bindings are in the list under the same header, `bindl`, as the media-key bindings that already work with the screen locked, and the count went 51, 53, then back to 51 after a reload. A control, `bindlx`, printed `bind: invalid flag`, so a refused line is visible. The first read of the binding list counted `^Bind` and reported 0, which was the pattern and not the list; the count is of `key:` lines. Whether a real lid event runs them was not measured: the VM has no lid.

## 2. The decision and the action

- [x] 2.1 Add the adapter verb that enables or disables a named output, with the reasoning at the verb and the header's list of verbs updated. Verify with the isolation check, and that a wrong state word, a missing name and a name the compositor does not have each fail with a reason and a non-zero status
- [x] 2.2 Add `gridline-lid` with its two arguments, the built-in panel prefixes, the external-screen test and the environment override for the connector tree, with the fail-safe branch at the top. Verify it contains no compositor name or command, by the project's isolation check
- [x] 2.3 Add the two lid-switch bindings to the compositor defaults. Verify they are in the binding list of the running compositor with the locked-session flavor, and that no other binding collides
- [x] 2.4 Register the program in the install manifest. Verify it is in the built package, executable, under a Gridline-named path, and that every build gate passes

Results:

- **2.1** The `output` verb is in `src/adapters/hyprland/adapter` and the header lists six verbs. On the VM, with a headless output as the target: no name, no state, a state that is neither word, a name with a space, a name with a comma and a name the compositor does not have each exit 1 with a reason and change nothing (2 active outputs before and after). `off` took the active count to 1 and `on` returned it to 2. The name check is not decoration: the compositor's own keyword printed `ok` for `NOSUCH-9, disable`, so without the check a mistyped name would succeed.
- **2.2** `src/session/gridline-lid`, with the fail-safe at the top and the panel prefixes `eDP`, `LVDS`, `DSI`. The adapter-isolation check passes on it. It was shown to scan the file by appending a `hyprctl` line to a copy: the check reported `src/session/gridline-lid references hyprctl`, and reported 0 references once restored. A pipeline hid the exit status in the first read, so the claim rests on the message and not on the status.
- **2.3** Installed on the VM, the running compositor picked the new defaults up on its own: 53 bindings (51 plus the two), `configerrors` empty, both under the `bindl` header with the media-key bindings, exactly one binding on `switch:on:Lid Switch`.
- **2.4** `gridline-lid` is in the built package, mode 755 under `/usr/libexec/gridline/`, and the six build gates pass. The adapter in the package is the new one.
- **Not exercised here:** a real lid event reaching the bindings. The VM has no lid switch, so what the bindings do on a lid event is group 4.

## 3. Driven where it can be

- [x] 3.1 Run `gridline-lid` against fake connector trees: panel and external both present with the lid closed, with the lid open, panel only with the lid closed, an unreadable tree, an argument that is neither word, and a panel name with each of the three prefixes and one without. Read the adapter calls it makes in each. Every case but the first must end in enable
- [x] 3.2 Drive the real adapter and program together on the VM against a headless output in place of the external screen: with the real tree, a fake tree naming a panel, and the compositor's own output as the panel, read the monitor list after closed and after open
- [x] 3.3 Verify a failure leaves the panel in use: make the adapter unavailable and call the program with `closed` and an external screen present, and read that the panel was not disabled and that the program says what failed

Results:

- **3.1** 17 cases against made-up connector trees and a recording stand-in for the adapter (a copy of the program with only its adapter directory changed). Only a lid reported `closed` with an external connector reading `connected` disables. Enabled in every other case: lid open, panel only, other connectors `disconnected` or `unknown`, an argument that is neither word (including `CLOSED` and none), and an external's status file missing. `eDP-1`, `LVDS-1` and `DSI-1` are each taken out, a panel with an unlisted name is never touched, two built-in panels are both taken out, and a tree with no panel or none at all makes no calls and exits 0. The adapter refusing and no adapter at all each exit 1 with a message and leave the panel as it is.
- **3.2** On the VM's live compositor with the installed program and adapter, and a headless output named `eDP-1` as the panel: with the real connector tree (one connected connector, no panel) `closed` changes nothing; with a tree naming the panel beside the real connected connector, `closed` took the panel out (active `Virtual-1`), `open` returned it (`Virtual-1 eDP-1`), calling `closed` twice left it out and `open` brought it back; with the external reading `disconnected` and the lid closed, the panel stayed.
- **3.3** With the program pointed at a directory holding no adapter and an external attached, `closed` exited 1, the panel stayed active, and the message was on stderr and in the journal under `gridline-lid`.

## 4. On the owner's laptop

Needs a person with the lid and the external monitor, and cannot be driven here. The defect was found there, and this is the check that it is gone.

- [ ] 4.1 Log in from the display manager with the lid closed and the external monitor attached. Verify the bar, the pointer and a terminal opened with the terminal key are on the external screen, and read the compositor's log for the lid-switch line at start-up
- [ ] 4.2 With the session running, open the lid and read that the panel comes back, then close it and read that it goes. Verify the windows that were on the panel are still reachable
- [ ] 4.3 Lock the session, close and open the lid, and verify the panel follows and the lock is still in place
- [ ] 4.4 Close the lid with no external monitor attached and read what the machine does, which the design expects to be suspend. Verify the session is intact on resume
- [ ] 4.5 Unplug the external monitor with the lid closed and record what happens. This answers the design's open question and decides whether the listener alternative is a follow-up

## 5. Coherence

- [x] 5.1 Update `docs/compositor-adapter.md` with the new verb, and `docs/user-experience.md` and `docs/recovery.md` with what a lid-closed laptop does and what to do if a screen stays dark. Verify every link resolves and nothing still says the laptop panel is left on
- [x] 5.2 Verify each scenario of the added requirement is exercised or recorded as unexercised with a reason, counting scenarios from the delta file rather than from the proposal's description of them
- [x] 5.3 Update the status in `README.md` and `website/index.html` to say what was and was not measured, and that group 4 is the owner's

Results:

- **5.1** `docs/compositor-adapter.md` has a section on the requests an adapter answers, with the new `output` verb and its two measured behaviors; `docs/user-experience.md` has a row and a section, "The lid", and `docs/recovery.md` has "A screen stays dark with the lid closed", with the text-console command that puts the panel back. Every in-page link resolves (`#the-lid`, `#settings`) and nothing says the panel is left on.
- **5.2** Six scenarios, counted from the delta file, none of which is exercised end to end here because none can be without a lid:
  1. *A session that starts with the lid closed*: the effect (panel out, windows and focus on the other screen) is driven on the VM in 3.2 and 1.2; the compositor's start-up report of the lid exists only in the owner's laptop log, and the whole is 4.1. Unexercised on the VM: the start-up event.
  2. *The lid closes during a session*: the program and verb driven in 3.2, the windows' move in 1.2; the event reaching the binding is 4.2. Unexercised: the event.
  3. *The lid opens*: 3.2 and 1.2 (the workspace came back); the event is 4.2.
  4. *Closing the lid with no other screen attached*: the policy is exercised in 3.1 and in 3.2 (c), where the panel stayed; that the machine then suspends is 4.4.
  5. *The lid is handled while the screen is locked*: the bindings carry the locked-session flavor, read in 2.3; a lid event at a lock screen is 4.3. Unexercised: the event.
  6. *A failure leaves the built-in screen in use*: exercised, 3.1 (adapter refusing, none installed, unreadable status, unreadable tree) and 3.3 on the live compositor.
- **5.3** `README.md` and `website/index.html` say the laptop was tried once, that the lid defect was found there, and that the fix is driven on the VM and not yet confirmed on it. The website has a `partial` item for it. They no longer say nothing has run on hardware.

## Workflow follow-up

- Run `/opsx:verify` once groups 1 to 3 are done. Group 4 stays open until the owner has run it, as `add-hardware-key-bindings` has, and the change is archived only after that.
- If 4.5 finds the system does not suspend, the event-listener alternative in the design becomes its own change.
