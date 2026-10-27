# Tasks

## 1. The adapter layer and the wire format

- [x] 1.1 Create `packages/gridline/src/adapters/` and verify the path matches the one `docs/compositor-adapter.md` and `docs/architecture.md` already name, by grep rather than by memory
- [x] 1.2 Document the wire format in `docs/compositor-adapter.md`, replacing the sentence that defers it to this change, and verify the document no longer says the format is owned by a later change
- [x] 1.3 Verify the documented format states all four parts the session relies on: the query argument, one capability per line on stdout, the meaning of exit status, and that an absent line and a non-zero exit are the same answer
- [x] 1.4 Verify the capability names in the documented format are character-for-character the names in the required and optional tables of the same document, since a format that renames a capability breaks the contract it implements

## 2. The Hyprland adapter

- [x] 2.1 Add the Hyprland adapter at `packages/gridline/src/adapters/hyprland/adapter`, executable, and verify the launcher's existing discovery glob finds it by running that glob against the staged tree
- [x] 2.2 Implement `capabilities` and `confirm`. `capabilities` reports only what installed files establish and exits non-zero when a file the adapter needs is absent; `confirm` asks the Wayland registry through `wayland-info` and reports each protocol-backed capability when the running compositor advertises its interface. Match each interface name as a whole word, so a longer name containing a shorter one cannot produce a false positive, and exit non-zero on output that cannot be parsed rather than reporting a partial set. `screencast` needs both halves: the screencopy protocols **and** a `.portal` file declaring `org.freedesktop.impl.portal.ScreenCast` with its D-Bus service installed. Verify by removing the compositor's portal backend and confirming `screencast` is absent from `confirm` even while the protocols are still advertised
- [x] 2.3 Implement the `hyprctl`-served capabilities under `confirm`, reporting them only when `hyprctl` answers against the running instance. Verify three states, not two: `hyprctl` absent, `hyprctl` present with no instance, and `hyprctl` answering a running compositor. The middle state is the one a present-or-absent test misses, and it is the state every pre-launch invocation is in
- [x] 2.4 Verify the adapter reports no capability it cannot deliver through a documented interface, by listing for each reported capability the interface that backs it and checking each against the design table
- [x] 2.5 Verify the adapter exits non-zero when it cannot produce a complete answer, and that it writes the reason to stderr rather than printing a partial capability list
- [x] 2.6 Verify the adapter checks structure and names independently, with two synthetic `wayland-info` outputs. One carries the expected structural markers and interface names the adapter does not look for: it must exit zero and report none of the protocol-backed capabilities, which is an empty answer about the compositor. The capabilities a working `hyprctl` establishes are still reported, since those do not come from the registry. The other carries unexpected markers and the interface names the adapter does look for: it must exit non-zero and report nothing, which is an unreadable answer about the tool. An adapter that matches names without checking structure passes the first and fails the second. Read the failure record in both, since both end the launch and only the record says which happened
- [x] 2.7 Verify the adapter's list of Wayland interface names matches the table in design.md character for character, by comparing the two rather than reading them. The list is hardcoded in the adapter because a shell adapter cannot read a design document at runtime, so it is a second copy of a claim the project records elsewhere, and this project has twice paid for a second copy nobody compared. Record which names appear in one and not the other, including none

## 3. The launcher invokes the adapter

- [x] 3.1 Replace the launcher's third branch with the two-phase sequence: `capabilities`, then `start`, then `confirm`. Verify the branch that reported "cannot invoke it" is gone by grep, and that the three invocations appear in that order
- [x] 3.2 Gate on `confirm`, not on `capabilities`: compare `confirm`'s answer against the required set and report the first missing capability by name through the existing `gridline-session` syslog identifier. Verify with a stub whose `confirm` omits one required capability, reading `journalctl -t gridline-session`, and verify separately that a capability absent from `capabilities` but present in `confirm` does not block, since absence before launch means not yet establishable
- [x] 3.3 Verify an adapter whose `confirm` exits non-zero is treated as reporting nothing, using a stub that exits 1 after printing some capabilities, and verify that a non-zero exit from `capabilities` ends the launch before `start` runs, by confirming no compositor process was created
- [x] 3.4 Start the compositor through the adapter rather than naming it in the launcher, and verify by grep that the launcher contains no compositor name, binary or protocol
- [x] 3.5 Verify an unreported optional capability does not block startup, using a stub that reports every required capability and neither optional one
- [x] 3.6 Verify the launcher's required-capability list matches the required table in `docs/compositor-adapter.md` character for character, by comparing the two rather than reading them. The list is a second copy for the same reason the adapter's interface names are, and it decides which launches are refused, so a name that drifts here refuses a session for a capability the contract never required. Record which names appear in one and not the other, including none

## 4. The isolation check

- [x] 4.1 Add the isolation check under `packages/gridline/tools/`, partitioning the tree by the documented adapter path, and verify it reports the partition sizes rather than passing silently on an empty scan
- [x] 4.2 Populate the reference list from what this change introduces: the compositor binary name, `hyprctl`, the protocol names in design.md, and the Hyprland configuration file name; verify each entry matches something that actually exists in the adapter
- [x] 4.3 Anchor the check in `debian/rules` beside the ownership gate, and verify both run before `dh_builddeb` by reading the build log for their output
- [x] 4.4 Verify the check fails on a planted leak, by temporarily placing a compositor reference in a file outside the adapter path, confirming the build fails naming that file and reference, and removing it. The same task that plants it removes it, and the removal is verified rather than assumed
- [x] 4.5 Verify a clean tree passes, and record the number of files scanned on each side of the boundary, since a check that scans nothing passes for the wrong reason

## 5. Packaging

- [x] 5.1 Record `wayland-utils` in `docs/component-ownership.md` and `docs/package-matrix.md` with its version, archive component and absence behavior, and verify the row states it is `universe` and not Hypr-ecosystem, so the existing claim that every Hypr-ecosystem package is in `universe` stays true rather than being stretched to cover it

- [x] 5.2 Declare `hyprland`, `xdg-desktop-portal-hyprland` and `wayland-utils` as `Depends` of `gridline-session`, and verify with `dpkg-deb -f` on the built package rather than by reading the control file
- [x] 5.3 State the `universe` support gap in the `gridline-session` long description, using the same sentence the live documents carry, and verify the wording is identical by comparing against `docs/component-ownership.md`
- [x] 5.4 Correct the two statements describing `gridline-session` as shipping no content, and verify the four describing `gridline-config` and `gridline-tools` are untouched, since those two still ship directories and no files
- [x] 5.5 Add the adapter layer to `debian/gridline-session.install` so the built package carries `usr/libexec/gridline/adapters/hyprland/adapter`, and verify with `dpkg -c` on the built package rather than by reading the manifest. Without this the launcher ships looking for a directory the package does not create, and fails on a real machine at the first branch while passing every test run from the source tree
- [x] 5.6 Build the packages and verify the ownership gate and the isolation check both pass, and that the adapter installs under `/usr/libexec/gridline/adapters/hyprland/adapter` with mode 755
- [x] 5.7 Remove `hyprland` and `xdg-desktop-portal-hyprland` with
  `apt-get remove --purge`, verify `apt-get autoremove` leaves them absent,
  then declare the dependency and confirm `apt` resolves it from the archive
  alone, since the measurement that informed the design installed both as
  manual packages and the baseline this task verifies is a clean install
## 6. The session starts

Execution note: this group needs a graphical console on the VM, not `lxc exec`. Open it with `lxc console gridline-vm --type=vga`, which needs a SPICE client on the host. The session either reaches a desktop or returns to the greeter, and both outcomes are observations this group records.

- [x] 6.1 Record the adapter's capability answer on the VM before selecting the session, and verify every required capability is reported, so a later failure is attributable to startup rather than to the capability check
- [ ] 6.2 Select Gridline at the greeter and verify a tiling desktop starts and accepts keyboard and pointer input
- [ ] 6.3 Verify the session is recorded as running for that user, by reading `loginctl` and confirming the desktop identity is the one the session entry sets rather than the compositor's own
- [ ] 6.4 Verify screen capture resolves to the compositor's portal backend in the running session, which is testable without a capturing application: read the portal frontend's own record of which implementation it selected for `org.freedesktop.impl.portal.ScreenCast` from `journalctl --user -u xdg-desktop-portal`, and confirm it names `org.freedesktop.impl.portal.desktop.hyprland`. Record that the match happened through `Hyprland` in the desktop identity rather than through `Gridline`, since `hyprland.portal` lists no `Gridline` in `UseIn` and `gridline-portals.conf` belongs to `add-desktop-portal-integration`. A reader who finds this working must not conclude the Gridline-owned path exists
- [ ] 6.5 Log out and verify the compositor terminates, the seat returns to the display manager, and the greeter is usable without a reboot
- [ ] 6.6 Verify the failure path still works after the success path exists, by removing the adapter and confirming the session reports the missing adapter and returns to the greeter, then restoring it by reinstalling the package and verifying both the file's md5 and its mode
- [ ] 6.7 Record which of the two optional capabilities the adapter reported on this machine, since the answer is hardware-dependent and a reader of the result needs to know what was and was not exercised
- [ ] 6.8 Decide whether `docs/component-ownership.md` gains a row for the compositor's own session entry, now that the greeter has been seen with both offered. Record the decision and its reason either way, since a question left open in a document outlives the change that opened it, and verify the section added by this change no longer says the answer is pending

## Workflow follow-up

- Run `/opsx:verify` once the session has been observed starting and the isolation check has run in a real build.
- Archive the change, then propose `add-desktop-portal-integration`, which owns the portal configuration file this change leaves to the frontend's own resolution.
