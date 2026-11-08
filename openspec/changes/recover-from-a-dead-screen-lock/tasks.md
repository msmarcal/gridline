# Tasks

## 1. Reproduce the state before changing anything

Nothing here can be verified against an accident. The first question is whether the state can be produced on purpose, and the second is whether killing a locker produces the same state as whatever happened the first time.

- [ ] 1.1 Kill the locker in a locked session and record what the screen shows, what `loginctl` reports, and what is in the journal. Compare against the incident: the compositor's fallback, `LockedHint=no`, and nothing recorded anywhere. A difference means the deliberate case is not the accidental one and the rest of this change is being verified against the wrong thing
- [ ] 1.2 Establish whether a locker started afterwards can attach, which the incident suggests and which the whole approach depends on. The shipped compositor defaults set `misc:allow_session_lock_restore = true`; verify that is what makes it possible, by measuring with the setting off as well as on
- [x] 1.3 **The machine could record a crash all along, and the first step of this task was built on a wrong measurement.** `core_pattern` pipes to apport, which holds seventeen crash reports on this VM, one of them written at 01:05 today during a Gridline session. The tool used to ask, `coredumpctl`, is not installed: every reading of "no cores captured" was a command-not-found read as an empty result, by me, repeatedly, including into this change's own design.

  With the right place to look, the evidence points away from a crash: **there is no report for the locker under any user**, nothing in apport's configuration excludes it, and other components of the same session do have reports. A locker that crashed would have left one.

  **So the limit, written down rather than guessed past:** the locker exited rather than crashed, and why it exited is unknown. No journal entry from it, nothing from the lock entry point, which does write its failures, and nothing in the compositor's log. What this rules out is the hypothesis everyone starts with, and that is worth more than the blank it replaces

## 2. The lock does not die with the thing that scheduled it

Found while mitigating the first incident, and it is the more reproducible of the two problems: stopping the idle unit killed the running locker and stranded the session. This is not an accident that needs waiting for.

- [ ] 2.0 Reproduce it deliberately: with a session locked, stop `gridline-idle.service` and confirm the locker dies with it. The journal of the first occurrence reads `Stopping gridline-idle.service`, then `swayidle: Terminated`, then the lock entry point's trap firing, which is a locker killed by the unit's cgroup going down rather than by anything about the locker
- [x] 2.1 **The lock is a unit now.** `gridline-lock.service` runs the entry point, and the idle daemon asks systemd to start it rather than running the locker itself, so the locker lives in its own cgroup and the daemon's lifetime is no longer the lock's. Read back from systemd: `ExecStart=/usr/libexec/gridline/gridline-lock`, `Restart=on-failure`, `StartLimitBurst=3`, and `systemd-analyze verify` reports nothing against either unit.

  **`Restart=on-failure` and not `always`, and the distinction is the behaviour rather than a preference.** A locker that exits zero was dismissed by the user who authenticated; restarting it would lock the screen they just unlocked. A locker that dies leaves the session held by the compositor with no client, which is the state this change exists to get out of. The same line therefore implements what tasks 3.1 and 3.2 ask for, and they are left open because neither is verified until a locker is actually killed.

  **Two things came with it that the task did not ask for and that are worth naming.** Starting an already-active unit is a no-op, so systemd now prevents a second locker stacking on the first, which the entry point had a `pgrep` guard for; the guard stays, narrowed to the case systemd does not cover, which is someone running the program by hand. And `before-sleep` stops blocking suspend: running the locker directly meant `swayidle` waited for it to exit before releasing the sleep inhibitor, which is to say the machine waited for the user to authenticate before suspending. Starting a unit returns once it has started
- [ ] 2.2 Verify the separation against the same three events rather than against the one that was noticed: a deliberate stop, a restart, and the unit failing
- [x] 2.3 **The suspicion was wrong, and finding that out is what stopped a change being made for the wrong reason.** The entry point calls `SetLockedHint` on `/org/freedesktop/login1/session/auto`, and that looked unreliable because it failed twice. Measured, the condition is simple and the object is fine:

  | Context | `auto` resolves |
  |---|---|
  | Inside the user manager, which is where the entry point runs | **yes**, to the display session |
  | A process with no session and outside the user manager | no |
  | A trap firing while its own unit is being stopped | **yes**, reproduced with a transient unit |

  `SetLockedHint` itself also succeeds from the user manager, not just the property read. So one of the two failures was my own invocation from outside any session, which was never a fair test, and the other was something narrower than the object not resolving.

  **What remains unexplained is one failure at one moment**, during the stop of the unit the locker was living inside. The likeliest remaining cause is the same one task 2.1 exists for: the call was made by a process in a cgroup that was being torn down, so the `busctl` it spawned was killed with everything else. If that is it, separating the lock's lifetime fixes this too and no change to the object is needed. **Re-check after 2.1 rather than changing the object now**

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
