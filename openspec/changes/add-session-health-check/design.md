# Design

## Context

See proposal.md for motivation. Four facts, all measured, and one of them corrects a measurement recorded in the change that produced it.

**Nothing reports a non-blocking gap.** `gridline-session` has one reporting path, `report_missing`, and all eight of its call sites end in `exit 1`. `gridline-tools` ships no executable at all: its package contains a directory and the files debhelper puts there.

**The hold is rate-limited, not unbounded.** `add-desktop-shell-components` recorded that `Upholds=` revives a unit past its own start limit. The manual says the opposite in plain words, that the usual per-unit rate limit applies, and the arithmetic agrees: `StartLimitBurst=5` within `StartLimitIntervalSec=10` permits at most 60 starts in two minutes and 59 were counted. What looked like a limit being ignored was the limit being honoured at a rate too high to notice. **The restart counter resetting, which is what suggested otherwise, is a counter reset and not a limit reset.**

**The platform has backoff and no "give up".** systemd 259 carries `RestartSteps` and `RestartMaxDelaySec`, which lengthen the interval between restarts. Nothing in `Upholds=` expresses stopping: the manual's own phrasing is that the units listed are started whenever found inactive **or failed**, continuously, to keep them running whatever happens.

**One machine already has the condition.** The verification VM cannot run the wallpaper daemon, for a reason established and confirmed elsewhere. The failing case does not have to be staged, which matters because a staged failure is a failure someone designed to be detectable.

## Goals / Non-Goals

**Goals:**

- A user who is missing a component can find out, by asking, what is missing and whether it matters.
- A component that cannot start stops being started, while one that stops after having worked is still restarted.
- The diagnostic answers when the session did not come up, which is the case it exists for.

**Non-Goals:**

- Changing which components are blocking. The launcher's refusal is a separate obligation and is correct.
- A general health framework. This reports the components the session declares, and nothing else observes anything.
- Watching components over time, alerting, or acting on what it finds. The diagnostic answers a question when asked.

## Decisions

### The session records; the command reports

The command does not inspect units other packages own. That coupling is what the project has avoided everywhere else, and it would make the diagnostic's correctness depend on the internals of `waybar.service` and its siblings.

So a component that is absent or fails leaves a record the session wrote, and `gridline doctor` reports from the record. The record is Gridline's own, which also makes it answerable when the session is not running at all: the failing case most worth diagnosing is the one where nothing is up to be inspected.

What the record contains is the Gridline name of the component, which of absent and failed it is, whether it is blocking, and when. Not a unit name: a reader asking why their bar is missing should not have to know that the bar is `waybar.service`.

### How a component stops being held is the part this design does not yet know

The decision is made and the mechanism is not. `Upholds=` has no "give up", so the candidates below each need measuring before one is chosen, and each is written here with the thing that would disqualify it.

| Candidate | What would disqualify it |
|---|---|
| Lengthen the interval with `RestartSteps` and `RestartMaxDelaySec` | It slows the loop rather than stopping it, which is not what was decided. It may still be worth doing underneath whatever stops it |
| A condition on a flag the wrapper writes after repeated failure, so systemd skips the unit cleanly | Whether `Upholds=` leaves a condition-skipped unit alone, or starts it again every cycle because it is inactive. Measurable in minutes |
| Have the session drop the `Upholds=` for a component that gave up | Needs something running to notice and act, which is a watcher, which is a non-goal |
| Have the wrapper refuse to run after N failures and exit successfully | A unit that exits 0 is inactive, and `Upholds=` starts inactive units too |

The second is the only candidate that is both cheap and plausibly correct, so it is the one to measure first. If it fails, this design is wrong about the mechanism and not about the decision, and that is the distinction to keep.

**Where the failure count lives matters more than it looks.** A component that failed five times in a session and a component that failed five times across a week are different conditions, and a count that survives logout would make a machine stop trying a component that failed once a day for a week. The count belongs to the session.

### The command exists because it is owed, not because this change wants a surface

`package-delivery` already assigns `gridline doctor` to `gridline-tools`, and that package was created to own a command surface it has never had. This change delivers the first one. The shape should therefore be the shape of the eventual command and not a one-off: `gridline doctor` reporting components is the first of several things the project has already said it will report, policy that is not in effect among them.

## Risks / Trade-offs

- **A diagnostic that reports from a record is only as good as the record.** → A component that fails in a way the session does not notice is a component the diagnostic will call healthy. The alternative, inspecting other packages' units, trades that for a coupling the project has refused elsewhere. The honest mitigation is that the record is written where the session already knows, which is the wrapper that starts each component.
- **Stopping a hold can hide a transient failure.** → A component that would have recovered on the sixth attempt is now a component the session gave up on. The count and the window are the whole of this trade-off, and both belong in the session rather than in a file that outlives it.
- **Correcting a measurement recorded in an archived change.** → The archived `tasks.md` says `Upholds=` revives past the start limit. It does not, and the corrected reading is here rather than in the archive. Nothing is edited there; what the archive records is what was believed when it was written, which is the point of an archive.

## Open Questions

- Whether `Upholds=` leaves a unit alone when its condition fails, or starts it again on every cycle. This decides the mechanism above and is measurable in minutes on the verification VM, which already has a component that cannot start.
- Where the record lives so that the diagnostic can read it when the session is not running, without it becoming state that outlives the session's own lifetime.
- Whether the diagnostic reports the compositor capabilities the adapter already knows about, which `compositor-abstraction` says are reported in a form the diagnostic commands can surface. That is a third promise of the same kind, and it is not in this change's scope, but the command's shape should not make it awkward to add.
