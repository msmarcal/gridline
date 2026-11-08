# Tasks

## 1. Reproduce the state before changing anything

Nothing here can be verified against an accident. The first question is whether the state can be produced on purpose, and the second is whether killing a locker produces the same state as whatever happened the first time.

- [ ] 1.1 Kill the locker in a locked session and record what the screen shows, what `loginctl` reports, and what is in the journal. Compare against the incident: the compositor's fallback, `LockedHint=no`, and nothing recorded anywhere. A difference means the deliberate case is not the accidental one and the rest of this change is being verified against the wrong thing
- [ ] 1.2 Establish whether a locker started afterwards can attach, which the incident suggests and which the whole approach depends on. The shipped compositor defaults set `misc:allow_session_lock_restore = true`; verify that is what makes it possible, by measuring with the setting off as well as on
- [ ] 1.3 Investigate why the first locker died, as far as the evidence allows, and **write down the limit rather than a guess**. Nothing recorded it on a machine whose `coredumpctl` captures nothing even for a crash systemd reports as dumped, so the first step is making the machine able to record a crash at all

## 2. The lock does not die with the thing that scheduled it

Found while mitigating the first incident, and it is the more reproducible of the two problems: stopping the idle unit killed the running locker and stranded the session. This is not an accident that needs waiting for.

- [ ] 2.0 Reproduce it deliberately: with a session locked, stop `gridline-idle.service` and confirm the locker dies with it. The journal of the first occurrence reads `Stopping gridline-idle.service`, then `swayidle: Terminated`, then the lock entry point's trap firing, which is a locker killed by the unit's cgroup going down rather than by anything about the locker
- [ ] 2.1 Separate the lock's lifetime from the unit that decides *when* to lock. A daemon that watches for inactivity and a locker that holds the screen have different lifetimes, and today the second lives inside the first's cgroup, so a stop, a package upgrade that restarts the unit, or `Upholds=` reacting to a failure all take the lock down with it
- [ ] 2.2 Verify the separation against the same three events rather than against the one that was noticed: a deliberate stop, a restart, and the unit failing
- [ ] 2.3 **`SetLockedHint` is called on an object that does not always resolve.** `gridline-lock` uses `/org/freedesktop/login1/session/auto`, and that failed twice in one session: from a context with no session of its own, and from the entry point's own trap while its unit was shutting down, logged as `could not set LockedHint to false`. The explicit session path works from the same context where `auto` fails, measured. It also succeeded at least once today, so `auto` is conditionally reliable and the condition is not understood: establish it, and resolve the session explicitly if the condition cannot be made dependable

## 3. A dead locker is replaced

- [ ] 3.1 Make the lock entry point replace a locker that dies, bounded, so that a session whose locker fails once returns to a password prompt without another machine. Verify by killing the locker and watching the prompt come back
- [ ] 3.2 Bound it, and verify the bound by making the locker impossible to start rather than by reading the code: the failure this must not produce is an invisible loop in the lock path
- [ ] 3.3 Verify that giving up is visible. A session that stops trying to lock itself, with nothing on screen and nothing in the journal, is the worst outcome available here and is worse than the loop
- [ ] 3.4 Verify the safety half still holds: no claim of a lock that no component is maintaining, and a session that stays locked while the compositor holds it. This is the property `platform-integration` already required and which held during the incident, and it is the one most likely to be broken by changing the entry point's exit paths

## 4. The recovery path is written down

- [ ] 4.1 Add a section to `docs/recovery.md` for a session that locks and will not unlock. It covers a session that does not start and says nothing about this, which is the failure that strands the user rather than returning them to the login screen
- [ ] 4.2 Name `swaylock` in it, and say plainly that the compositor's own fallback screen names a different program that Gridline does not ship. A user reading that screen is being sent after something not installed, and the project cannot change the screen, only make its own instructions correct
- [ ] 4.3 Verify the written path by following it from a second machine, exactly as written, against a session in the failed state. A recovery path that has not been walked is a draft

## 5. Coherence

- [ ] 5.1 Correct the screen locker row in `docs/component-ownership.md`, whose fallback cell describes a session that refuses to start. That is what happens when the locker is absent at startup; it is not what happens when the locker dies later, and the row covers only the first
- [ ] 5.2 Verify each scenario of the modified requirement is exercised or recorded as unexercised with a reason, counting scenarios from the delta file rather than from the proposal's description of them

## Workflow follow-up

- Run `/opsx:verify` once a killed locker has been recovered from on the VM and the written path has been walked.
- This change does not depend on `add-session-health-check` and does not block it; they touch different components and the same documents in different places.
