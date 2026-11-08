# Proposal

## Why

Four live documents name `add-ci-vm-smoke-tests` as the owner of automated verification, one of them with a table row reading "Re-runs the gate in CI". The change has never existed, as a change or an archive entry, and there is no CI of any kind in this repository.

What stands in for it is a person. Every session-level verification this project has done was a human running commands and reading output: the effective `ExecStart` of each unit, the target's `After=` and the journal of the right manager instance for ordering cycles, a component killed and returning with its configuration intact, a session started with three binaries moved aside, the Ubuntu session carrying no failed unit, a record agreeing with the machine, a diagnostic answering with no session at all. None of that survives a contributor, and the project has said it wants contributors.

The four build gates do run, and they work: the most recent one caught three deliberately planted failures. They run when someone builds the package, which on this project has always been the same person who wrote the thing being checked.

## What Changes

**This change ships implementation.**

- Runs the package-level checks without a person: the build, the four gates, the contents and metadata of each produced package, conffile registration, and `systemd-analyze verify` on every unit and drop-in Gridline ships.
- Runs the session-level checks that need no display: the target's wiring and ordering, each unit's effective `ExecStart`, the start program's decisions against planted and absent configuration, restart after a kill, giving up after a start limit, and the diagnostic answering with no session.
- Runs on the GitHub replica, because that is where a runner exists. The project's own remote is self-hosted and has none.

**Out of scope, and recorded as a question rather than dropped.** The graphical layer: the compositor starting, the bar drawing, a notification appearing, the wallpaper, the lock screen. Those need a display, and the cheap route does not work: this project measured a headless compositor failing with `Cannot open backend: no allocator available`, because its backend needs a GPU allocator rather than merely no monitor. The known path is a virtual KMS device, which is a kernel module needing a privileged `modprobe` and a kernel built with it, which is why projects that do this run a VM with their own kernel rather than loading a module on the runner. GitHub's Linux x86-64 runners do expose `/dev/kvm`, with a permissions step, so it is possible and it is not cheap. Deciding it needs evidence this change is meant to produce, not precede.

**The name says VM and this change ships no VM.** It is kept because four documents point at it and because the VM is the part deferred rather than the part abandoned.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

- `package-delivery`: gains one requirement. The capability already requires that certain conditions be caught, and its scenarios say "during build or review". Review is a person, and a build is whoever ran it, so both are satisfied today by the author of the change checking their own work. What is missing is that the checks run somewhere the author does not control, which is the difference between a guard and a habit.

## Impact

- A CI workflow, in whatever form the GitHub replica accepts, which does not exist in any form today.
- `packages/gridline/tools/`: possibly a single entry point the workflow and a developer both call, so that what CI runs and what a person runs are the same thing rather than two lists that drift.
- `docs/packaging.md`: the row promising that the gate is re-run in CI, which becomes true.
- `docs/architecture.md`, `docs/compositor-adapter.md`, `docs/package-matrix.md`: the other three references to this change.
- Nothing in the session packages changes. This change adds verification and alters no behaviour.
