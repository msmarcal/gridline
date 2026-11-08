# Design

## Context

See proposal.md for motivation. What is already here, and what the measurements say about the part being deferred.

**Four gates exist and run at package build**, anchored at `execute_before_dh_builddeb`: path ownership, adapter isolation, Gridline-named declared paths, and the drop-in role matching its unit. They are small shell programs with their own reasoning in comments, and each was shown failing against a planted defect before being relied on.

**No CI exists.** The project's own remote is self-hosted with no runner. A replica is pushed to GitHub, which is where a runner can be had.

**The graphical layer was measured, not assumed.** A compositor started with its headless backend on this project's own VM failed with `Cannot open backend: no allocator available`: the backend wants a GPU allocator, so "no monitor" is not the same as "no display hardware". The known answer is a virtual KMS device, a kernel module that needs a privileged `modprobe` and a kernel built with it, which is why the projects that test compositors this way run a VM with their own kernel rather than loading a module on a runner. GitHub's Linux x86-64 runners do expose `/dev/kvm`, with a permissions step, and not on ARM64, where the host does not surface it at all.

## Goals / Non-Goals

**Goals:**

- A change that breaks a gate says so without anyone choosing to check.
- A contributor's local run and the automated run are the same checks, from one definition.
- What nobody is checking is written down.

**Non-Goals:**

- The graphical layer. Deferred with evidence, in this file, rather than omitted.
- Replacing human verification of what a person has to see. The value of automating the rest is to leave attention for that.
- A test framework. These are the project's existing checks, run somewhere else, plus the ones a session can answer without a screen.

## Decisions

### One entry point, two callers

The workflow and a developer call the same program. If the workflow carries its own list of steps, the two drift, and the first symptom is a contributor whose machine is green and whose push is red for a reason neither list explains.

So the checks live in `packages/gridline/tools/`, where the existing four already live, behind one entry point that runs them in order and reports which failed. The workflow's job is to install dependencies and call it.

### Two layers, and the line between them is a display

| Layer | What it answers | What it needs |
|---|---|---|
| Package | the gates, what each package contains and declares, conffile registration, `systemd-analyze verify` on every shipped unit and drop-in, what the maintainer scripts do on install and removal | a container |
| Session without a display | the target's wiring and ordering, each unit's effective `ExecStart`, the start program's decisions against planted and absent configuration, restart after a kill, giving up after a start limit, the diagnostic with no session | a container with a systemd user manager |

The second layer is the one worth stating plainly, because it is larger than it sounds. Most of what this project has verified by hand never needed a screen: whether a unit's `ExecStart` resolves to what was intended, whether a target's `Wants=` still names one blocking service, whether a component killed comes back carrying its configuration, whether a start limit is honoured, whether a record agrees with the machine. Those are the checks that caught real defects, and none of them is a screenshot.

### Proving each check can fail

Every check added here is shown failing against a deliberately broken input before it is relied upon. The project has done this twice already and both times it mattered: a gate whose reference list was empty would have passed everything, and a gate reading a manifest's second field rather than its third would have reported nothing wrong.

This is in the delta as a scenario rather than left as a convention, because it is the one practice that separates a check from a decoration.

### What the deferred layer needs before it is decided

Not a preference, a measurement: whether a compositor can be brought up on a runner at all. That is one experiment, and it either produces a working display or produces the reason it cannot, and either outcome settles the question. It is a task here and it gates nothing else.

## Risks / Trade-offs

- **A green run that proves less than it appears to.** → The scenario requiring that unchecked things be named exists for this. The risk is not a check that fails; it is a suite that passes while nobody notices that the session's appearance is unverified by anything but a person.
- **CI on a replica rather than on the project's own remote.** → The replica is where a runner exists. It means a push to the self-hosted remote is unverified until it reaches the replica, and the delay is worth stating where contributors read it.
- **Checks that drift from what they guard.** → One definition, two callers. The failure this avoids is subtle and expensive: a workflow quietly testing an older shape of the project.

## Open Questions

- Whether a compositor can be started on a hosted runner, and at what cost. The cheap route is already disqualified by measurement; the remaining routes are a virtual KMS device inside a VM, or a different compositor for the test than the one shipped, which would be testing something else.
- Whether the session layer runs in a container with a user manager or needs a VM for `systemd --user` to behave as it does on a real machine. This is measurable before any of it is written, and it decides how much of the second layer is reachable cheaply.
