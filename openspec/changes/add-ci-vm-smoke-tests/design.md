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

### What the deferred layer needed, and what the measurement said

Not a preference, a measurement: whether a compositor can be brought up on a runner at all. One experiment, settling the question either way.

It ran, and the answer came back in two halves that this section had expected to be one.

**A display device is obtainable, and the cost this project had recorded for it was wrong.** The runner already carries `/dev/dri/card1` before anything is loaded, it has passwordless sudo, and `vkms` ships in the Azure kernel's module tree. `modprobe vkms` loads and `card0` appears. The route written down here as needing a VM with its own kernel needs neither.

**And the compositor aborts on it anyway**, with `CBackend::create() failed!`, a core dump and no Wayland socket. So the headless backend fails for want of an allocator, and a real DRM device fails at backend creation. What a runner cannot give is not a card; it is whatever the backend wants behind one.

The graphical layer therefore stays out, now for a measured reason rather than a presumed cost. `.github/workflows/display-probe.yml` is kept rather than deleted, because an answer this specific can expire: the compositor, the kernel or the runner image could each turn it over, and re-asking is one dispatch.

## Risks / Trade-offs

- **A green run that proves less than it appears to.** → The scenario requiring that unchecked things be named exists for this. The risk is not a check that fails; it is a suite that passes while nobody notices that the session's appearance is unverified by anything but a person.
- **CI on a replica rather than on the project's own remote.** → The replica is where a runner exists. It means a push to the self-hosted remote is unverified until it reaches the replica, and the delay is worth stating where contributors read it.
- **Checks that drift from what they guard.** → One definition, two callers. The failure this avoids is subtle and expensive: a workflow quietly testing an older shape of the project.

## Open Questions

Both questions this document opened with were answered by measurement and are recorded as decisions above. Neither answer was the one the question expected, which is why they are worth restating here rather than quietly deleted.

The compositor question assumed the remaining route was "a virtual KMS device **inside a VM**". The device loaded directly on the runner, with no VM; what failed was the compositor's backend. The cost was in a different place than the question put it.

The session-layer question asked whether a container suffices or a VM is needed for `systemd --user` to behave. A container suffices, and every behaviour this project depends on was correct there - `Wants=` and `Upholds=` reported apart, the implicit `After=`, `ExecStart` through a drop-in, `Restart=`, a start limit reached. The whole prerequisite is `loginctl enable-linger`, without which there is no user manager and no `XDG_RUNTIME_DIR` at all, because a job runs as a user with no login session.

What is left open is not about this change:

- **Nothing verifies that the build still calls the entry point.** `debian/rules` invokes `run-checks`, and the workflow invokes it separately so that losing one does not lose both. Neither notices if the other stops.
- **Task 4.4's first half is answerable only where there is a display.** It was measured on the project's VM and cannot be measured on a runner, because the components exit the moment they find no screen. The session-layer check names it as unanswerable on the machine it runs on, which stays correct.
