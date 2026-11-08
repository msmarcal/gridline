# Tasks

## 1. Reproduce the state before changing anything

Nothing here can be verified against an accident. The first question is whether the state can be produced on purpose, and the second is whether killing a locker produces the same state as whatever happened the first time.

- [x] 1.1 **Reproduced on purpose, and it matches the accident where it should and differs where the fix had already landed.** Stopping the lock unit with the session locked: zero lockers, `LockedHint=no`, and the compositor still holding the session, which is the incident exactly.

  **One difference, and it is this change's doing rather than a mismatch.** The journal now says `Main process exited, code=exited, status=143` and `Failed with result 'exit-code'`. The incident recorded nothing anywhere, because the locker was not a unit then and systemd had nothing to report about it. Making the lock a unit gave the failure a place to be seen, which is most of what task 3.3 asks for.

  **An ordering note worth keeping.** This task was written to capture the baseline before changing anything, and the baseline was captured by the accident instead: the fix for task 2.1 went in first, because the design named it the prerequisite. The deliberate reproduction above is therefore of the post-fix code producing the pre-fix state, which it still does when the unit is stopped rather than when its locker dies
- [x] 1.2 **A locker started afterwards attaches, measured four times now** rather than inferred from the incident: twice while recovering a stranded session by hand, once by the unit's own `Restart=on-failure` after the locker was killed, and once by starting the unit again immediately after stopping it. Each time the password prompt returned and the session unlocked normally.

  **Not measured: the same thing with `misc:allow_session_lock_restore` turned off.** That half would show the setting is what makes it possible rather than something else, and it cannot be done while a session is locked without deliberately producing a state nothing can recover from the screen. It needs its own arranged run with the compositor keyword flipped while unlocked, and it is recorded as owed rather than assumed
- [x] 1.3 **The machine could record a crash all along, and the first step of this task was built on a wrong measurement.** `core_pattern` pipes to apport, which holds seventeen crash reports on this VM, one of them written at 01:05 today during a Gridline session. The tool used to ask, `coredumpctl`, is not installed: every reading of "no cores captured" was a command-not-found read as an empty result, by me, repeatedly, including into this change's own design.

  With the right place to look, the evidence points away from a crash: **there is no report for the locker under any user**, nothing in apport's configuration excludes it, and other components of the same session do have reports. A locker that crashed would have left one.

  **So the limit, written down rather than guessed past:** the locker exited rather than crashed, and why it exited is unknown. No journal entry from it, nothing from the lock entry point, which does write its failures, and nothing in the compositor's log. What this rules out is the hypothesis everyone starts with, and that is worth more than the blank it replaces

## 2. The lock does not die with the thing that scheduled it

Found while mitigating the first incident, and it is the more reproducible of the two problems: stopping the idle unit killed the running locker and stranded the session. This is not an accident that needs waiting for.

- [ ] 2.0 Reproduce it deliberately: with a session locked, stop `gridline-idle.service` and confirm the locker dies with it. The journal of the first occurrence reads `Stopping gridline-idle.service`, then `swayidle: Terminated`, then the lock entry point's trap firing, which is a locker killed by the unit's cgroup going down rather than by anything about the locker
- [x] 2.1 **The lock is a unit now.** `gridline-lock.service` runs the entry point, and the idle daemon asks systemd to start it rather than running the locker itself, so the locker lives in its own cgroup and the daemon's lifetime is no longer the lock's. Read back from systemd: `ExecStart=/usr/libexec/gridline/gridline-lock`, `Restart=on-failure`, `StartLimitBurst=3`, and `systemd-analyze verify` reports nothing against either unit.

  **`Restart=on-failure` and not `always`, and the distinction is the behaviour rather than a preference.** A locker that exits zero was dismissed by the user who authenticated; restarting it would lock the screen they just unlocked. A locker that dies leaves the session held by the compositor with no client, which is the state this change exists to get out of. The same line therefore implements what tasks 3.1 and 3.2 ask for, and they are left open because neither is verified until a locker is actually killed.

  **Two things came with it that the task did not ask for and that are worth naming.** Starting an already-active unit is a no-op, so systemd now prevents a second locker stacking on the first, which the entry point had a `pgrep` guard for; the guard stays, narrowed to the case systemd does not cover, which is someone running the program by hand. And `before-sleep` stops blocking suspend: running the locker directly meant `swayidle` waited for it to exit before releasing the sleep inhibitor, which is to say the machine waited for the user to authenticate before suspending. Starting a unit returns once it has started
- [x] 2.2 **Intact through all three**, and the locker's PID was compared rather than its presence, because a locker that died and was replaced looks identical to one that never died if you only count processes.

  | Event on `gridline-idle.service` | Locker |
  |---|---|
  | `systemctl stop` | PID 1166717, unchanged |
  | `systemctl restart` | PID 1166717, unchanged |
  | `systemctl kill --signal=KILL` | PID 1166717, unchanged |

  The first of those is the one that stranded a user earlier today, in the same session, on the same machine. `LockedHint` read `yes` throughout, set by the lock unit at start, which also answers the part of task 2.3 that was still open in the normal case
- [x] 2.3 **The suspicion was wrong, and finding that out is what stopped a change being made for the wrong reason.** The entry point calls `SetLockedHint` on `/org/freedesktop/login1/session/auto`, and that looked unreliable because it failed twice. Measured, the condition is simple and the object is fine:

  | Context | `auto` resolves |
  |---|---|
  | Inside the user manager, which is where the entry point runs | **yes**, to the display session |
  | A process with no session and outside the user manager | no |
  | A trap firing while its own unit is being stopped | **yes**, reproduced with a transient unit |

  `SetLockedHint` itself also succeeds from the user manager, not just the property read. So one of the two failures was my own invocation from outside any session, which was never a fair test, and the other was something narrower than the object not resolving.

  **What remains unexplained is one failure at one moment**, during the stop of the unit the locker was living inside. The likeliest remaining cause is the same one task 2.1 exists for: the call was made by a process in a cgroup that was being torn down, so the `busctl` it spawned was killed with everything else. If that is it, separating the lock's lifetime fixes this too and no change to the object is needed. **Re-check after 2.1 rather than changing the object now**

## 3. A dead locker is replaced

- [x] 3.1 **Killed with `SIGKILL` and replaced**: PID 1166806 became 1166831, the unit reports `NRestarts=1`, and `LockedHint` stayed `yes` throughout. Where this session showed the compositor's fallback earlier today, with nothing to type into, a password prompt comes back on its own.

  The clean-exit path was measured in the same run and behaves the other way, which is the distinction `Restart=on-failure` exists for: after the user authenticated, the unit went inactive with `Result=success` and `NRestarts=0`, and nothing locked the screen they had just unlocked
- [x] 3.2 **Bounded, verified by making the locker impossible to start rather than by reading `StartLimitBurst`.** With the locker moved aside: `NRestarts=3`, the unit reached `failed` with `Result=exit-code`, and stopped. No loop.

  The test was safe and that is a property of the design rather than luck: the entry point checks the locker before locking, so an unstartable locker never takes the screen. `LockedHint=no` and zero lockers throughout, and the session stayed usable while its own lock was failing three times.

  Restored afterwards and checked three ways: checksum equal to the one taken before, mode `755 root:root`, and `dpkg --verify` clean
- [x] 3.3 **Visible in the journal, which is the bar this task sets**, and it names the reason rather than only the failure: `gridline-lock: /usr/bin/swaylock is not installed`, then the unit failing, then `Failed to start gridline-lock.service`. The outcome this task guards against, nothing on screen and nothing in the journal, does not happen.

  **And the diagnostic contradicts it, which is a different defect and is recorded rather than folded in.** While the lock had given up three times, `gridline doctor` reported `locker  running`. The record for that component is written once by the launcher at session start and nothing updates it, because the locker has no unit of Gridline's with an `ExecStopPost` the way the desktop shell components do. The same is true of the polkit agent, which crashed in this very session while the diagnostic reported it running.

  That belongs to `report-components-that-stop`, which exists for it. Keeping it out of this change is deliberate: the polkit agent has nothing to do with screen locking, and a fix riding along inside an unrelated change is how this project once carried a false claim in a package description across two changes
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
