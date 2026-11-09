# Tasks

## 1. Before anything is built

Every earlier component here was measured before it was configured, and each time the measurement changed what got built. The workaround was only tried by hand, with a window opened and closed. The design rests on two claims that nobody has driven.

- [ ] 1.1 Record the failure as a person meets it, with the session unlocked: open the launcher, type Wi-Fi, choose it, and read the compositor's client list and the program's own stderr. This is the baseline that task 3.1 is the opposite of
- [ ] 1.2 Establish that the launcher hands its own `PATH` to what it starts: put a probe ahead on `PATH` for one launcher only, start an entry whose `Exec` names it, and confirm the probe ran and not the system's file. The whole design rests on this
- [ ] 1.3 Measure what the real application does at launch: how fast it refuses, with what status, and that it stays up when it is accepted. Those numbers set the failure window and are not to be guessed
- [ ] 1.4 Open every top-level panel of the settings application in turn with the workaround applied by hand, capture each, and classify it: opens with content, opens empty or with an error, or writes a setting nothing here applies. Record the list, and for each panel in the third class the process that would normally apply it and that it is not running

## 2. The stand-in

- [ ] 2.1 Add the stand-in under `/usr/libexec/gridline/shims/` with the settings application's name. Verify it runs the real program by absolute path, appends `GNOME` to the existing identity rather than replacing it, and adds nothing else, by reading the running panel's environment from `/proc`
- [ ] 2.2 Implement the failure window and the notification, with the constant and its reason at the top. Verify both directions: make the real path unusable and read that a notification names the failure, and start it normally and read that none appears
- [ ] 2.3 Make `gridline-menu apps` put that directory first on `PATH` for the launcher it starts, in a Gridline session only. Verify from the launcher's own environment that it is there, and in a session with another identity that it is not
- [ ] 2.4 Register it in the install manifest. Verify it is in the built package, executable, under a Gridline-named path, and that every build gate passes

## 3. Measured as a person reaches it

Needs the session unlocked, and a person to unlock it: the launcher takes keyboard focus and a locked session holds it.

- [ ] 3.1 Drive the real launcher: open it, type Wi-Fi, press Return, and read which window the compositor lists. Do the same for Settings, Bluetooth and Network. Each is the opposite of the 1.1 baseline
- [ ] 3.2 Verify the identity is changed for nothing else: start another program from the same launcher and read its environment, then the launcher's, then the user manager's. All three must still name Gridline first and none may name GNOME
- [ ] 3.3 Verify another session is unaffected: run the launcher role with another desktop's identity and confirm it adds nothing to `PATH`, so the settings application starts as it would have without Gridline
- [ ] 3.4 Verify absence degrades: move the real program aside and choose a settings entry. The launcher must stay up, a notification must say it did not open, and the session must be unaffected

## 4. Coherence

- [ ] 4.1 Add settings to `docs/user-experience.md` and `docs/component-ownership.md`, which mention neither. State what works, what 1.4 found does nothing, and that typing the command in a terminal still fails because only the launcher is covered
- [ ] 4.2 Add a check to the session-layer check that the stand-in's name still matches the program the real package ships. Verify it fails when the stand-in is renamed, and that where the application is not installed it names itself as unanswerable rather than passing
- [ ] 4.3 Verify each scenario of the added requirement is exercised or recorded as unexercised with a reason, counting scenarios from the delta file rather than from the proposal's description of them

## Workflow follow-up

- Run `/opsx:verify` once group 3 has been driven and group 1's panel classification is written down.
- If 1.4 finds that several panels do nothing, that is a follow-up change and not a reason to hold this one: reaching Wi-Fi and Bluetooth is the requirement, and the classification is what keeps the rest from being a surprise.
