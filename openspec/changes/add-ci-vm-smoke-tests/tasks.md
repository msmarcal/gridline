# Tasks

## 1. The measurement that sizes the second layer

The package layer is known to be reachable in a container. The session layer is not, and how much of it is reachable decides how much of this change is worth writing.

- [ ] 1.1 Establish whether a `systemd --user` manager behaves in a container as it does on a real machine, for the things this project checks: `Wants=` and `Upholds=` reported separately, the implicit `After=` a target gains, `ExecStart` resolved through drop-ins, `Restart=` honoured, and a start limit reached. Measure each rather than assuming the manager is the manager
- [ ] 1.2 Record what the container cannot answer, with the reason, before writing anything that depends on it. A list of what is unreachable is the input to the scenario that requires unchecked things to be named

## 2. One entry point

- [ ] 2.1 Add a program in `packages/gridline/tools/` that runs the project's checks in order and reports which failed, and make the package build call it rather than listing the four gates itself. Verify the build still fails when a gate fails, by planting a failure
- [ ] 2.2 Verify a developer and the automated run call the same thing, by reading both: the workflow's job is to install dependencies and call the entry point, and it carries no list of its own

## 3. The package layer

- [ ] 3.1 Build the packages and run the gates. Verify the run fails when a gate fails, with a planted defect, rather than only confirming it passes today
- [ ] 3.2 Check what each package contains and declares: the paths, the dependencies read from the built package rather than from `debian/control`, and conffile registration compared against the package's own `/etc` contents entry by entry
- [ ] 3.3 Run `systemd-analyze verify` on every unit and drop-in Gridline ships. Verify it catches a deliberately malformed one, because this is the check most likely to be added for the look of it. A directive in the wrong section is the case to plant: this project shipped `StartLimitBurst` in `[Service]`, where systemd ignores it, and nothing noticed
- [ ] 3.4 Exercise install and removal: the session entries diverted and given back, the shell units disabled and restored, and the record of what was disabled. Verify against a machine state before and after rather than against the scripts' own output

## 4. The session layer, as far as it reaches

- [ ] 4.1 Verify the session target's wiring: `Wants=` naming the blocking components and the shell target, `Upholds=` naming the idle daemon, and the launcher's blocking check reading one of them. This is the shape a change can break without any test noticing today
- [ ] 4.2 Verify each component's effective `ExecStart` resolves through Gridline's drop-in to the start program with the right role, read from systemd rather than from the file on disk
- [ ] 4.3 Verify the start program's decisions: no desktop identity gives the bare binary, a Gridline session with nothing of the user's gives Gridline's paths, a planted user configuration gives the bare binary, and an unreadable shipped default gives the bare binary. Four cases, read from the process that replaced it
- [ ] 4.4 Verify a component killed returns with its configuration, and a component that cannot start stops being started after its limit. Both are behaviour this project changed deliberately and neither has anything watching it
- [ ] 4.5 Verify the diagnostic answers with no session at all, and that its exit codes are what they claim, measured without a pipe
- [ ] 4.6 Verify everything in this group fails when it should, by planting a defect for each rather than by observing a green run

## 5. Running it where a runner exists

- [ ] 5.1 Add the workflow to the GitHub replica and verify it runs on a push, reporting a failure in a way that reaches whoever would merge
- [ ] 5.2 Verify the failure report names the check and what it found, not just that something exited non-zero. A red mark with no name sends a contributor to read the whole log
- [ ] 5.3 Record in the project's documentation that verification happens on the replica and not on the self-hosted remote, including the delay that implies, where a contributor reads it rather than where a maintainer does

## 6. What nobody is checking

- [ ] 6.1 Write down what the automated run does not cover and why, naming each: the compositor starting, the bar drawing, a notification appearing, the wallpaper, the lock screen, and the judgements that need a person, such as whether a value on screen is current rather than merely present
- [ ] 6.2 Establish whether a compositor can be started on a hosted runner at all, and at what cost. The cheap route is disqualified by measurement already: a headless backend failed with `Cannot open backend: no allocator available`. Produce either a working display or the reason there cannot be one, and record which
- [ ] 6.3 Correct `docs/packaging.md`, whose table promises the gate is re-run in CI, and the three other live documents that name this change. Verify by grep that no live document still describes this as planned

## 7. Coherence

- [ ] 7.1 Verify each scenario of the added requirement is exercised or recorded as unexercised with a reason, counting scenarios from the delta file rather than from the proposal's description of them
- [ ] 7.2 Verify the entry point is itself covered: a check that fails to run is indistinguishable from a check that passes, unless the run says how many checks it executed

## Workflow follow-up

- Run `/opsx:verify` once the workflow has reported a failure for a planted defect and a pass for the tree as it stands.
- This change touches no session behaviour, so it can land alongside the others rather than after them.
