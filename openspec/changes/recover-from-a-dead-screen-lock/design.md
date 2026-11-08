# Design

## Context

See proposal.md for motivation. What is known about the incident, and what is not, because the difference decides what this change may claim.

**Known.** The session locked on idle. The locker stopped running. The compositor kept the session locked, which is what `ext_session_lock_v1` does when the locking client dies, and showed its own fallback screen. That screen has no password field and instructs the user to run `hyprlock`, which this project does not ship. `gridline-lock` exited with its locker and its `EXIT` trap cleared `LockedHint`, so the system's view read unlocked while the session was locked, which is the direction `platform-integration` already requires. Starting a fresh locker afterwards worked: it attached, accepted the password, and the desktop came back, so `misc:allow_session_lock_restore = true` in the shipped compositor defaults is doing its job.

**Not known, and not guessed at here.** Why the locker died. No journal entry from the locker, nothing from `gridline-lock`, which does write its failures, and nothing in the compositor's log.

**One thing that was known turned out to be measured wrongly**, and the correction matters because it changes where to look. This design first recorded that the machine captures no cores. It does: `core_pattern` pipes to apport, which has been collecting crash reports throughout, including one written during a Gridline session while this change was being planned. What was absent was the tool used to ask, `coredumpctl`, which is not installed here, so every "no cores" reading was a command-not-found read as an empty result.

With the right place to look, the evidence says something: **there is no crash report for the locker, under any user, and nothing in apport's configuration excludes it.** Other components have reports, including one from the same session. So the locker very likely did not crash; it exited. Why a locker would exit on its own is a different question from why one would crash, and it is the one still open.

**A second way to lose the locker, found while mitigating the first and more reproducible than it.** Stopping `gridline-idle.service` killed the running locker and stranded the session: the journal reads `Stopping gridline-idle.service`, then `swayidle: Terminated`, then the lock entry point's trap firing. The entry point runs as a child of the idle daemon, inside that unit's cgroup, so the unit going down takes the lock with it. A deliberate stop did it here; a package upgrade that restarts the unit, or `Upholds=` reacting to a failure, would do the same. **This one does not need an unexplained crash to happen.**

**`SetLockedHint` is called on an object that does not always resolve.** The entry point uses `/org/freedesktop/login1/session/auto`. It failed twice in one session, once from a context with no session of its own and once from the entry point's own trap during its unit's shutdown, and it succeeded at least once earlier the same day. The explicit session path works from the same context where `auto` fails, measured. So `auto` is conditionally reliable and the condition is not understood, which is a different statement from either "it works" or "it is broken".

**Why this is a change rather than a bug report.** The locker is a blocking component precisely because a desktop that cannot lock must not run. That reasoning covers the machine being left open. It does not cover the machine being left shut, and this incident is the second kind: the session was protected throughout and its owner was locked out of it.

## Goals / Non-Goals

**Goals:**

- A locker that dies is replaced, so that someone at the keyboard can authenticate.
- A user who has to recover by hand finds instructions that name what they actually have.
- The safety property that already holds keeps holding: no claim of a lock nobody maintains, and a session that stays locked.

**Non-Goals:**

- Preventing the locker from dying. Unknown cause, and a component that cannot fail is not something this project can build.
- Choosing a different locker. That decision is recorded elsewhere and nothing here reopens it.
- Watching the locker in general. This is about one failure with one response, not a supervisor.

## Decisions

### Separating the lock's lifetime from the idle daemon's comes first

A daemon that decides *when* to lock and a program that *holds* the screen have different lifetimes, and today the second lives inside the first's cgroup. Everything else in this design is built on a locker that can outlive the thing that scheduled it, so this is the prerequisite rather than a second improvement: replacing a dead locker is worth little if stopping an unrelated unit kills the replacement too.

What that means mechanically is open. The lock could be its own unit that the idle daemon starts rather than runs, or the entry point could detach from the cgroup it was invoked in. The first is more systemd-shaped and makes the lock startable by anything, which is the same concern the restart decision below raises.

### Replacing the locker, not supervising it

The entry point runs one locker and exits when it does. The smallest change that meets the requirement is for it to stay and start another, with a bound: a locker that cannot start at all must not become the loop this project just removed from somewhere else.

Three shapes, and the trade-off is the same one in each: how long a user stares at a screen with nothing to type into.

| Shape | What it costs |
|---|---|
| The entry point restarts its locker a bounded number of times | All of it stays in one small program that already owns the lock's lifetime. Nothing supervises it in turn, so if the entry point itself dies the problem returns |
| A systemd user unit with `Restart=` | systemd owns the restarting and the bound, which it is better at than a shell loop. The lock becomes a unit that can be started by anything, and a lock entry point that anything can invoke is worth thinking about before shipping |
| The compositor restarts it | Compositor-specific, so it belongs in the adapter, and it makes the lock's reliability a property of which compositor is in use |

The first is the one to measure first: smallest, and it keeps the lock's lifetime where it already is.

**The bound matters more than the restart.** A locker that dies once should be replaced. A locker that cannot start at all should stop being replaced and should leave something on screen or in the journal saying so, because an invisible loop in the lock path is how a machine becomes unusable without anyone learning why.

### The documentation has to name what is installed

The screen the user actually reads at the moment of failure is the compositor's, and it names `hyprlock`. Gridline cannot change that screen and should not try. What it can do is make its own documentation correct and findable, so that a user who goes looking finds a path that names `swaylock` and works.

This is the same shape as every other place the project writes down what the baseline does rather than arguing with it.

## Risks / Trade-offs

- **A restart loop in the lock path is worse than the thing it fixes.** → The bound is not optional, and the measurement that matters is what happens when the locker can never start, not what happens when it dies once.
- **Replacing a dead locker could mask the reason it died.** → It already is masked: nothing recorded the first one. The investigation is a task here, and if it finds a cause the fix for that cause belongs wherever the cause is.
- **Documentation for a failure nobody can reproduce on demand.** → The locker can be killed deliberately, which produces the same state for a known reason, and that is how the path gets exercised.

## Open Questions

- Why the locker died. Investigated as a task; its answer does not gate the rest of this change.
- Whether a deliberately killed locker produces exactly the state the incident produced, which decides whether the recovery path can be verified at all without waiting for a second accident.
