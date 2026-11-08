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
- [x] 3.4 **Holds, measured as one sequence rather than as four separate observations**, because the property is about the transitions and not about any single state:

  | | locker | `LockedHint` |
  |---|---|---|
  | unlocked | 0 | `no` |
  | locked | 2 | `yes` |
  | **lock unit stopped, compositor still holding** | 0 | **`no`** |
  | recovered | 2 | `yes` |

  The third row is the requirement: with the session locked and no component maintaining the lock, the system does not claim one. That is the direction that leaves the session protected, because something reading the hint concludes unlocked and may lock again, never the reverse.

  This was the property most likely to be broken by moving the entry point into a unit, since the clearing happens on its exit paths and those changed

## 4. The recovery path is written down

- [x] 4.1 **`docs/recovery.md` gains a section for it**, as a sibling of the path for a session that does not start rather than a note inside it, because the two failures leave the user in opposite places: one at the login screen, the other at a screen with nothing to type into.

  It explains what they are looking at and why the session stays locked, gives the steps, says why they should not need them now that the lock is a unit that replaces a locker which dies, and ends with terminating the session as the last step rather than the first, naming what that costs
- [x] 4.2 Named, and **the screen is wrong in a second way that only showed up on reading it again**. It tells the user to press `Ctrl+Alt+F[N]` with the terminal number it prints, and it printed `tty 0` for a session measured on `tty2`: following it sends someone to a terminal that does not exist. So the written path depends on neither claim, and says why: it asks `loginctl list-sessions` for the number instead of trusting the screen, and names `swaylock` rather than the locker the screen suggests.

  Gridline cannot change that screen. What it can do is make its own instructions correct and say plainly which parts of the other ones to ignore
- [ ] 4.3 Verify the written path by following it from a second machine, exactly as written, against a session in the failed state. A recovery path that has not been walked is a draft

## 5. Coherence

- [x] 5.1 **The row now describes both failures**, and its ownership cell stopped saying "integration only", which became false when Gridline took ownership of the unit that runs the locker and the entry point every lock request arrives through.

  **Correcting it broke a count, and the count was the interesting part.** The page's reading section asserts that every component falls into exactly one of three fallback categories. The locker now falls into two: absent at startup it blocks the session, and dying mid-session is a different failure with a different answer. Forcing the cell back into one category would have been the easy fix and would have reproduced the omission that left a user locked out with the documentation silent. The reading section says so instead, and the blocking table still lists the same six components
- [x] 5.2 **Eight scenarios, counted from the delta file, and counting them is what caught the error this change was built on.**

  | Scenario | State |
  |---|---|
  | Manual lock requires authentication | Inherited. Exercised: the lock started through the unit, the password was accepted, the session returned |
  | The session locks itself when idle | Inherited, **not exercised here**: the idle daemon was masked for most of this work as a mitigation, and the one idle lock that did fire is the incident this change exists for |
  | The system agrees that the session is locked | Inherited. Exercised: `LockedHint=yes` while locked, through every lock in this change |
  | A dead lock component is not reported as a session that is safe to treat as unlocked | Inherited. Exercised in 3.4: hint `no` with the compositor still holding, which is the protected direction |
  | Lock holds across suspend | Inherited, **not exercised**: no suspend was performed. Worth noting that this change altered that path, since `before-sleep` no longer blocks on the locker exiting |
  | Lock failure does not expose the session | Inherited. Exercised in 3.2: the locker was unstartable, the unit failed three times, and the screen never locked and never exposed anything it was holding |
  | A dead locker is recoverable without another machine | Inherited, and **this is the one the shipped code did not satisfy**. Exercised in 3.1: killed, replaced, prompt returned, and the written path in 4.1 covers the cases the unit cannot |
  | The recovery path names the locker the session actually uses | Added here. Exercised in 4.2, which also found that the compositor's screen misreports the terminal number |

  **The proposal claimed nothing required recoverability and that was false.** The scenario has been in `platform-integration` since `add-gridline-configuration-defaults`, and it was missed by reading the requirement through a `head -24` that cut it off at the fourth scenario. A second scenario in the delta duplicated it and was removed; the delta adds one scenario rather than two.

  That changes what this change is, which is why it is corrected in the proposal rather than quietly here: a gap in the specification is something to design, and a requirement the code did not meet is something that should have been caught by what the project runs, and was not

## Workflow follow-up

- Run `/opsx:verify` once a killed locker has been recovered from on the VM and the written path has been walked.
- This change does not depend on `add-session-health-check` and does not block it; they touch different components and the same documents in different places.
