# Tasks

## 1. The measurements the design left open

- [ ] 1.1 Establish how the adapter can tell whether the user has a compositor configuration of their own, using the component's own search paths rather than a guess at them. Record the paths and where they came from, and record what the check does when it cannot tell, since that case decides whether a user ever loses their own configuration
- [ ] 1.2 Establish what dpkg does with a conffile the package stops shipping, separately for an unmodified copy and an edited one. They differ, and the edited case is the one that matters to an administrator. Verify by exercising both on the VM rather than by reading policy
- [ ] 1.3 Confirm the compositor honours the explicit path by observing the running compositor, not by the command exiting zero. A path it reads and a path it ignores both succeed

## 2. The defaults move to a path Gridline names

- [ ] 2.1 Have the adapter's `start` verb name the configuration path, and only when the user has none of their own. Verify both branches on the VM: with a user configuration present the compositor must load theirs, with none it must load Gridline's, each read back from the running compositor
- [ ] 2.2 Stop shipping the loader at the compositor's own path, and stop `gridline-config` declaring `/etc/xdg/hypr/`. Verify with `dpkg -L` on the built package that neither appears, and on the VM that the directory is gone after an upgrade
- [ ] 2.3 Verify the keymap and the rest of the defaults still take effect on a first login, since this change moves how they are reached and must not change what they are. Compare a known value read from the running compositor against the one the previous arrangement produced
- [ ] 2.4 Verify the marker still reaches the user's terminal, since it is set by the defaults and the defaults are now reached differently
- [ ] 2.5 Verify both gates pass and account for the ownership gate's file count, which must fall by the loader this change stops shipping

## 3. The defaults reach no session but Gridline's

- [ ] 3.1 Verify the finding this closes actually failed before, by confirming from the archived record what a session started outside Gridline inherited. A fix whose before state was never observed is a fix nobody can check
- [ ] 3.2 Verify a compositor session started outside Gridline now resolves configuration as it would on a machine without Gridline, reading what the compositor loaded rather than inferring it from the absence of our keybindings
- [ ] 3.3 Close the open finding in `docs/compositor-adapter.md`, rather than leaving a document describing a problem that no longer exists. Verify by grep that no live document still says Gridline's defaults reach other sessions

## 4. A gate for Gridline-named declared paths

- [ ] 4.1 Add a check that fails when an install manifest declares a path no segment of which carries the project's name. Verify by planting such a path and confirming the build fails, since every path in the tree today passes and a gate that has never failed is not known to work
- [ ] 4.2 Verify the gate accepts the shapes the requirement blesses: a Gridline-named directory, and a Gridline-named file in a directory another component owns. Both exist in the tree and both must pass, or the gate is stricter than the requirement
- [ ] 4.3 Verify the gate would have caught the path this change removes, by planting it again and confirming the failure names it. That is the claim this change makes about why the violation shipped, and it should be demonstrated rather than asserted
- [ ] 4.4 Record in the check's own comments what it does not cover, in the same place the other two gates record theirs

## 5. The administrator who edited the file

- [ ] 5.1 Exercise the upgrade path for an unmodified conffile that the new version stops shipping, and record what dpkg does with it
- [ ] 5.2 Exercise the same upgrade with the conffile edited, and record what dpkg does and what the administrator has to do about it
- [ ] 5.3 Record the answer where an administrator would meet it, not only in this change, since whoever edited that file will not read an archived task list

## 6. Coherence

- [ ] 6.1 Verify each scenario of the modified requirement is exercised or recorded as unexercised with a reason, counting them from the delta file rather than from the proposal's description of them
- [ ] 6.2 Verify the rewritten scenario about a user's own configuration is satisfied by the new mechanism rather than by the old wording surviving, since a MODIFIED block replaces the requirement and the scenario's name stayed while its meaning changed
- [ ] 6.3 Update `docs/configuration.md`, whose section on what Gridline does about first-file-wins describes the mechanism this change replaces. Verify by grep that no live document still describes the loader
- [ ] 6.4 Update `docs/package-matrix.md` for the owned paths that change, and verify the list matches `dpkg -L` on the built packages rather than the previous text

## Workflow follow-up

- Run `/opsx:verify` once a session has started from the greeter with the defaults reached the new way, and once a compositor session outside Gridline has been observed.
- Archive, then resume `add-desktop-shell-components`, whose group 3 waits on the pattern this change settles.
