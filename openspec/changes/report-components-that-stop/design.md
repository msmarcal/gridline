# Design

## Context

See proposal.md for motivation. The mechanism as it stands, because the fix depends on which half is missing.

**Two writers, and they do not cover the same set.** The record under the session's runtime directory is written by the start program, which knows a desktop shell component is absent before it replaces itself with it, and by an `ExecStopPost` in the Gridline drop-in on each of those three units, which is the only thing left that knows how one ended. The launcher writes a record for the two blocking components at session start and never again, because neither has a Gridline drop-in: the polkit agent's unit is the archive's and untouched, and the screen locker had no unit at all until `recover-from-a-dead-screen-lock` gave it one.

**So the asymmetry is not about blocking.** It is about which components Gridline attached something to. The lock now has a unit of Gridline's own, which is the obvious place to put the missing half for it. The agent does not, and the project's rule is that influencing a third party's unit is done with a Gridline-named drop-in in a directory that supports one, which is exactly the shape already used three times.

**What the diagnostic claims today.** `running` for anything whose last record says `started`. The record carries `at=`, so the diagnostic already has what it needs to know how old its information is, and does not use it.

## Goals / Non-Goals

**Goals:**

- A blocking component that stops after the session started stops being reported as running.
- The diagnostic distinguishes what it has observed from what it is assuming.

**Non-Goals:**

- Watching components. The record is written by the components' own lifecycle events, not by anything polling.
- The platform integrations Gridline does not start. Recorded elsewhere as a different problem.
- Acting on why the agent crashed. It was a non-goal to find out, and it turned up anyway while checking the crash's timestamp: `SIGABRT` at 16:53:51, five seconds after the compositor aborted at 16:53:46. The agent did not crash on its own, it followed the compositor down. Whether that is worth preventing is somebody else's change; this one only had to notice it had happened.

## Decisions

### The same shape that already works, applied to the two that lack it

A Gridline-named drop-in on the agent's unit with an `ExecStopPost`, and the equivalent on the lock's own unit. Nothing new is invented: this is the third and fourth use of a mechanism already verified on three components, and the build gate that checks a drop-in starts the component its unit names already exists to guard the shape.

The alternative considered was having the diagnostic ask systemd for each component's state at report time. It is fewer files and it is the coupling this project refuses everywhere: the diagnostic's correctness would depend on the internals of units other packages own, and on their names, which is what the record exists to avoid.

### Both ends, and both writers, because neither knows the whole of it

This was an open question - the unit's `ExecStopPost` or the entry point it runs - and the answer is both, each for the one thing only it knows.

The entry point knows the locker is not installed, and can name the path it looked at, which is a different answer from a failure and sends a reader somewhere else. It also knows the moment the screen is about to be held, which is the only moment at which *running* is true of a component that spends most of a session not running. The unit knows there was an end at all, including when the program never ran: measured, `ExecStopPost` is reached for a main process that exits zero, one that exits non-zero, one killed with `SIGKILL`, and a unit whose `ExecStart` does not exist (status 203), with `SERVICE_RESULT`, `EXIT_CODE` and `EXIT_STATUS` set in every one. That last case is the lock giving up, which is one of the two defects this change exists to fix.

**Both ends and not just the stopping one.** With `ExecStopPost` alone the record goes started, stopped, and stays there: the component comes back, systemd restarts it, and the diagnostic reports a running component as stopped for the rest of the session. That was measured on the agent and is the same defect pointing the other way, worse in one respect, because it sends a reader to the journal for a component that is fine and teaches them to stop believing the report. The agent's unit therefore carries an `ExecStartPost` too. The other four components need none: the three shell ones start through a program of Gridline's that records before replacing itself, and the lock's entry point does the same.

### Not every component the session expects is meant to be running

The record needed a word it did not have. The bar is meant to be running, and a bar that is not running is a bar that is missing. The screen locker is not: it runs while the screen is locked and is absent from an unlocked desktop that is working perfectly.

Recording the locker's clean exit as `stopped` would have the diagnostic announce a missing blocking component on every working machine, which is the confidently wrong answer this change exists to stop producing. So there is a `ready` state, for installed and able to run and not running because nothing is asking, and a `finished` verb that differs from `stopped` in one thing only: what a clean exit means. Failure means the same for both.

### A component that is not installed did not fail

Last writer wins throws away the better answer. Measured: the locker was moved aside, the entry point recorded `absent` naming the path, the unit's `ExecStopPost` overwrote it with `failed`, and the diagnostic reported a failure for a component that was not there. The requirement is explicit that the report says which of absent and failed it is, so the fix for one defect was producing another against the same requirement.

`absent` is therefore not overwritten by a stop, and does not get stuck either: an explicit record replaces it, which is what the starting program writes the next time the component is actually there.

### Severity belongs to the component, not to the event that ends it

The launcher knows which components refuse a session. A unit's `ExecStopPost` knows only that its service stopped. So an unset `GRIDLINE_COMPONENT_BLOCKING` carries forward whatever the last record said, and only an explicit value overrides it.

Defaulting to `no` instead would have shipped a defect inside the fix for another one: the authentication agent is recorded as blocking by the launcher, and the drop-in added here records every one of its stops, so the diagnostic would have said that losing it merely degrades the desktop - in the same breath as finally getting its state right.

### A record has an age and the diagnostic should use it

Even with both halves, a record is a statement about the last event, not about now. A component killed with `SIGKILL` in a way that skips `ExecStopPost`, or a machine where the drop-in did not apply, leaves the diagnostic believing something stale.

So the report distinguishes what it observed from what it assumes. What that looks like was open, and the answer is the time, said per component, plus one line about whether a Gridline session is running at all.

**The time needs its date more often than it looks.** A first version printed the hour and minute only. A record from the previous day then read as "as of 04:00", which a reader takes for four o'clock this morning, and a record stamped in the year 2099 read as "as of 00:00", the most ordinary line on the page. Both were this command presenting a record it could not vouch for as a current one, which is the defect it exists to stop making - found by writing records on purpose that could not say when, rather than by reading the code. So the date appears whenever the record is not from today, a time later than now is named as being in the future, and a missing or unreadable one is said in words rather than hidden.

**Whether a session is running is said first, not last**, because it changes how every line below should be read. Gridline's own target answers it, and no unit another package owns is touched. A report from a session that has ended is history rather than status.

**The honest version of this is small and matters more than it looks.** A diagnostic that says "running, last seen at 01:23" is useful. One that says "running" about a component that died twenty minutes ago is worse than one that says nothing, because the reader stops looking.

## Risks / Trade-offs

- **A drop-in on the agent's unit is a fourth file modifying a third party's unit.** → The project already does this three times, the rule it follows is in the project context, and the alternative is coupling the diagnostic to unit internals.
- **`ExecStopPost` does not run for every death.** → Measured on a throwaway unit rather than quoted from the manual, because the whole approach rests on it: it is reached for a clean exit, a non-zero exit, a `SIGKILL`, and an `ExecStart` that does not exist. What skips it is the manager itself dying, which was *not* measured and is the case the record's age covers rather than pretends about.
- **Fixing a diagnostic nobody has relied on yet.** → It has been relied on twice today, by me, to decide whether something was running, and it was wrong both times.

## Open Questions

Both questions this document opened with are answered above, by measurement rather than by argument, and are recorded as decisions: the lock records from its unit *and* its entry point, and the diagnostic expresses age as a time with a date whenever the record is not from today.

What is left open is not about this mechanism:

- **The record is written in sessions that are not Gridline's.** `gridline-shell` records a component as started even when it execs it outside a Gridline session, so `gridline doctor` run from a GNOME session can read a partial set of records. The header added here mitigates it, because Gridline's target is not active there and the report says so, but the writing itself predates this change and belongs to whatever fixes it.
- **A `systemctl stop` of the lock while the screen is locked records `ready`.** systemd reports a requested stop as success even when the main process was killed, measured, so the clean-exit path is taken for a session that is in fact still locked by the compositor with no client. `recover-from-a-dead-screen-lock` carries the lockout itself; this is only the record agreeing with the wrong half of it.
- **The launcher's `ready` at session start is unexercised.** It needs a fresh login, and every measurement here ran inside one session.
