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
- Finding out why the agent crashed.

## Decisions

### The same shape that already works, applied to the two that lack it

A Gridline-named drop-in on the agent's unit with an `ExecStopPost`, and the equivalent on the lock's own unit. Nothing new is invented: this is the third and fourth use of a mechanism already verified on three components, and the build gate that checks a drop-in starts the component its unit names already exists to guard the shape.

The alternative considered was having the diagnostic ask systemd for each component's state at report time. It is fewer files and it is the coupling this project refuses everywhere: the diagnostic's correctness would depend on the internals of units other packages own, and on their names, which is what the record exists to avoid.

### A record has an age and the diagnostic should use it

Even with both halves, a record is a statement about the last event, not about now. A component killed with `SIGKILL` in a way that skips `ExecStopPost`, or a machine where the drop-in did not apply, leaves the diagnostic believing something stale.

So the report distinguishes what it observed from what it assumes. What that looks like is open: naming the time, or marking a record older than the session's start as unverified, or saying plainly which components the session is able to follow and which it is not. The requirement is that a reader can tell the difference, not that it be expressed one way.

**The honest version of this is small and matters more than it looks.** A diagnostic that says "running, last seen at 01:23" is useful. One that says "running" about a component that died twenty minutes ago is worse than one that says nothing, because the reader stops looking.

## Risks / Trade-offs

- **A drop-in on the agent's unit is a fourth file modifying a third party's unit.** → The project already does this three times, the rule it follows is in the project context, and the alternative is coupling the diagnostic to unit internals.
- **`ExecStopPost` does not run for every death.** → A process killed with `SIGKILL` still runs `ExecStopPost`, because systemd runs it after the main process is gone; what skips it is the manager itself dying. That is the case the record's age covers rather than pretends about.
- **Fixing a diagnostic nobody has relied on yet.** → It has been relied on twice today, by me, to decide whether something was running, and it was wrong both times.

## Open Questions

- Whether the lock's record should be written by the unit's own `ExecStopPost` or by the entry point it runs, which already knows when it is starting and exiting and already sets `LockedHint` at both moments.
- How the diagnostic expresses the difference between observed and assumed, and whether the record's `at=` is enough on its own.
