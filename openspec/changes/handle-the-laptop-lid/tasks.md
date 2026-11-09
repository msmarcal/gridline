# Tasks

## 1. Before anything is built

The design rests on claims taken from one log on one laptop and on how this compositor's output keyword behaves. Each is checked on the VM where it can be, and what cannot be is named, so that the group that is left for the owner's laptop is a short list and not a surprise.

- [ ] 1.1 Create a headless output on the VM's running compositor to stand in for an external screen, disable and re-enable it with the compositor's own output keyword, and read the monitor list after each. Record the keyword forms that work, including the one that returns an output to its preferred mode, position and scale
- [ ] 1.2 With two outputs and a window on the one that is then disabled, read where the window and its workspace are afterwards. The design expects the compositor to move them; this reads it
- [ ] 1.3 Disable an output, then make the compositor reload its configuration, and read whether the output is back. The design expects it not to survive; this reads it
- [ ] 1.4 Confirm on the VM that the connector names in `/sys/class/drm` and the compositor's output names are the same strings, and read what the VM's one output is called, so the naming claim is checked somewhere other than the laptop it came from
- [ ] 1.5 Load the two lid-switch bindings into the running compositor and read its configuration errors and its binding list, so a syntax the compositor refuses is found before it is shipped

## 2. The decision and the action

- [ ] 2.1 Add the adapter verb that enables or disables a named output, with the reasoning at the verb and the header's list of verbs updated. Verify with the isolation check, and that a wrong state word, a missing name and a name the compositor does not have each fail with a reason and a non-zero status
- [ ] 2.2 Add `gridline-lid` with its two arguments, the built-in panel prefixes, the external-screen test and the environment override for the connector tree, with the fail-safe branch at the top. Verify it contains no compositor name or command, by the project's isolation check
- [ ] 2.3 Add the two lid-switch bindings to the compositor defaults. Verify they are in the binding list of the running compositor with the locked-session flavor, and that no other binding collides
- [ ] 2.4 Register the program in the install manifest. Verify it is in the built package, executable, under a Gridline-named path, and that every build gate passes

## 3. Driven where it can be

- [ ] 3.1 Run `gridline-lid` against fake connector trees: panel and external both present with the lid closed, with the lid open, panel only with the lid closed, an unreadable tree, an argument that is neither word, and a panel name with each of the three prefixes and one without. Read the adapter calls it makes in each. Every case but the first must end in enable
- [ ] 3.2 Drive the real adapter and program together on the VM against a headless output in place of the external screen: with the real tree, a fake tree naming a panel, and the compositor's own output as the panel, read the monitor list after closed and after open
- [ ] 3.3 Verify a failure leaves the panel in use: make the adapter unavailable and call the program with `closed` and an external screen present, and read that the panel was not disabled and that the program says what failed

## 4. On the owner's laptop

Needs a person with the lid and the external monitor, and cannot be driven here. The defect was found there, and this is the check that it is gone.

- [ ] 4.1 Log in from the display manager with the lid closed and the external monitor attached. Verify the bar, the pointer and a terminal opened with the terminal key are on the external screen, and read the compositor's log for the lid-switch line at start-up
- [ ] 4.2 With the session running, open the lid and read that the panel comes back, then close it and read that it goes. Verify the windows that were on the panel are still reachable
- [ ] 4.3 Lock the session, close and open the lid, and verify the panel follows and the lock is still in place
- [ ] 4.4 Close the lid with no external monitor attached and read what the machine does, which the design expects to be suspend. Verify the session is intact on resume
- [ ] 4.5 Unplug the external monitor with the lid closed and record what happens. This answers the design's open question and decides whether the listener alternative is a follow-up

## 5. Coherence

- [ ] 5.1 Update `docs/compositor-adapter.md` with the new verb, and `docs/user-experience.md` and `docs/recovery.md` with what a lid-closed laptop does and what to do if a screen stays dark. Verify every link resolves and nothing still says the laptop panel is left on
- [ ] 5.2 Verify each scenario of the added requirement is exercised or recorded as unexercised with a reason, counting scenarios from the delta file rather than from the proposal's description of them
- [ ] 5.3 Update the status in `README.md` and `website/index.html` to say what was and was not measured, and that group 4 is the owner's

## Workflow follow-up

- Run `/opsx:verify` once groups 1 to 3 are done. Group 4 stays open until the owner has run it, as `add-hardware-key-bindings` has, and the change is archived only after that.
- If 4.5 finds the system does not suspend, the event-listener alternative in the design becomes its own change.
