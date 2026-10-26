# Tasks

## 1. The adapter layer and the wire format

- [ ] 1.1 Create `packages/gridline/src/adapters/` and verify the path matches the one `docs/compositor-adapter.md` and `docs/architecture.md` already name, by grep rather than by memory
- [ ] 1.2 Document the wire format in `docs/compositor-adapter.md`, replacing the sentence that defers it to this change, and verify the document no longer says the format is owned by a later change
- [ ] 1.3 Verify the documented format states all four parts the session relies on: the query argument, one capability per line on stdout, the meaning of exit status, and that an absent line and a non-zero exit are the same answer
- [ ] 1.4 Verify the capability names in the documented format are character-for-character the names in the required and optional tables of the same document, since a format that renames a capability breaks the contract it implements

## 2. The Hyprland adapter

- [ ] 2.1 Add the Hyprland adapter at `packages/gridline/src/adapters/hyprland/adapter`, executable, and verify the launcher's existing discovery glob finds it by running that glob against the staged tree
- [ ] 2.2 Implement the `capabilities` query, reporting each Wayland-protocol capability only when the compositor advertises that protocol, and verify against the protocol names recorded in design.md on the baseline
- [ ] 2.3 Implement the `hyprctl`-served capabilities, reporting them only when `hyprctl` is executable and answers, and verify by running the query with `hyprctl` present and with it renamed away
- [ ] 2.4 Verify the adapter reports no capability it cannot deliver through a documented interface, by listing for each reported capability the interface that backs it and checking each against the design table
- [ ] 2.5 Verify the adapter exits non-zero when it cannot produce a complete answer, and that it writes the reason to stderr rather than printing a partial capability list

## 3. The launcher invokes the adapter

- [ ] 3.1 Replace the launcher's third branch with an invocation of the adapter's `capabilities` query, and verify the branch that reported "cannot invoke it" is gone by grep
- [ ] 3.2 Compare the reported set against the required set and report the first missing capability by name through the existing `gridline-session` syslog identifier, verified by running the launcher against an adapter stub that omits one capability and reading `journalctl -t gridline-session`
- [ ] 3.3 Verify an adapter that exits non-zero is treated as reporting nothing, by running the launcher against a stub that exits 1 after printing some capabilities and confirming the session does not start
- [ ] 3.4 Start the compositor through the adapter rather than naming it in the launcher, and verify by grep that the launcher contains no compositor name, binary or protocol
- [ ] 3.5 Verify an unreported optional capability does not block startup, using a stub that reports every required capability and neither optional one

## 4. The isolation check

- [ ] 4.1 Add the isolation check under `packages/gridline/tools/`, partitioning the tree by the documented adapter path, and verify it reports the partition sizes rather than passing silently on an empty scan
- [ ] 4.2 Populate the reference list from what this change introduces: the compositor binary name, `hyprctl`, the protocol names in design.md, and the Hyprland configuration file name; verify each entry matches something that actually exists in the adapter
- [ ] 4.3 Anchor the check in `debian/rules` beside the ownership gate, and verify both run before `dh_builddeb` by reading the build log for their output
- [ ] 4.4 Verify the check fails on a planted leak, by temporarily placing a compositor reference in a file outside the adapter path, confirming the build fails naming that file and reference, and removing it. The same task that plants it removes it, and the removal is verified rather than assumed
- [ ] 4.5 Verify a clean tree passes, and record the number of files scanned on each side of the boundary, since a check that scans nothing passes for the wrong reason

## 5. Packaging

- [ ] 5.1 Declare `hyprland` and `xdg-desktop-portal-hyprland` as `Depends` of `gridline-session`, and verify with `dpkg-deb -f` on the built package rather than by reading the control file
- [ ] 5.2 State the `universe` support gap in the `gridline-session` long description, using the same sentence the live documents carry, and verify the wording is identical by comparing against `docs/component-ownership.md`
- [ ] 5.3 Correct the two statements describing `gridline-session` as shipping no content, and verify the four describing `gridline-config` and `gridline-tools` are untouched, since those two still ship directories and no files
- [ ] 5.4 Build the packages and verify the ownership gate and the isolation check both pass, and that the adapter installs under `/usr/libexec/gridline/adapters/hyprland/adapter` with mode 755
- [ ] 5.5 Install on the baseline VM and verify the declared dependencies resolve from the archive alone, with no third-party source configured

## 6. The session starts

Execution note: this group needs a graphical console on the VM, not `lxc exec`. Open it with `lxc console gridline-vm --type=vga`, which needs a SPICE client on the host. The session either reaches a desktop or returns to the greeter, and both outcomes are observations this group records.

- [ ] 6.1 Record the adapter's capability answer on the VM before selecting the session, and verify every required capability is reported, so a later failure is attributable to startup rather than to the capability check
- [ ] 6.2 Select Gridline at the greeter and verify a tiling desktop starts and accepts keyboard and pointer input
- [ ] 6.3 Verify the session is recorded as running for that user, by reading `loginctl` and confirming the desktop identity is the one the session entry sets rather than the compositor's own
- [ ] 6.4 Verify screen capture reaches the compositor backend, since `screencast` is the capability whose absence would have blocked startup and whose presence is otherwise untested
- [ ] 6.5 Log out and verify the compositor terminates, the seat returns to the display manager, and the greeter is usable without a reboot
- [ ] 6.6 Verify the failure path still works after the success path exists, by removing the adapter and confirming the session reports the missing adapter and returns to the greeter, then restoring it by reinstalling the package and verifying both the file's md5 and its mode
- [ ] 6.7 Record which of the two optional capabilities the adapter reported on this machine, since the answer is hardware-dependent and a reader of the result needs to know what was and was not exercised

## Workflow follow-up

- Run `/opsx:verify` once the session has been observed starting and the isolation check has run in a real build.
- Archive the change, then propose `add-desktop-portal-integration`, which owns the portal configuration file this change leaves to the frontend's own resolution.
