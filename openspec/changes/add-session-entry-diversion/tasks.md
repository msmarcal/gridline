# Tasks

## 1. The measurements the design left open

- [ ] 1.1 Establish the right `dpkg-divert` shape for a displacement with no replacement file. Gridline supplies nothing at the diverted path and only wants the original gone, which is not the usual case the tool is written for. Verify by applying the candidate form and reading `dpkg-divert --list`, then confirming the login screen no longer offers the entry
- [ ] 1.2 Establish what the coverage check enumerates against: the compositor package's file list as the archive carries it, or the entries present on the installed machine. They answer different questions, and the check is worthless if it answers the one nobody asked. Record which and why
- [ ] 1.3 Confirm, from the installed compositor package, every file it ships under the session entry directory, so the diversion covers a measured set rather than the two this change happens to know about

## 2. The diversion

- [ ] 2.1 Add a `postinst` to `gridline-session` registering a diversion for every session entry the compositor ships. Verify with `dpkg-divert --list` that each is registered and with `ls` that the original path no longer holds the entry
- [ ] 2.2 Add a `postrm` removing every diversion the package registered. Verify by removing the package and confirming `dpkg-divert --list` is clean and each file is back at its original path
- [ ] 2.3 Verify the compositor package still verifies after the diversion, with `dpkg --verify` on it reporting nothing, since a diversion relocates a file rather than altering it and anything else would mean the wrong mechanism was used
- [ ] 2.4 Verify an upgrade of the compositor package while the diversion is in place leaves the diversion intact and the new file at the diverted path, by reinstalling that package and re-reading both
- [ ] 2.5 Verify both build gates still pass and that the ownership gate's file count is unchanged, since this change ships maintainer scripts rather than files into a package

## 3. Covering every entry, now and later

- [ ] 3.1 Add a check that every session entry the compositor ships is covered by a diversion, using whichever enumeration 1.2 settled. Verify by planting an uncovered entry and confirming the check reports it, since a check that has never failed is not known to work
- [ ] 3.2 Verify the check runs where it will be seen, and record which stage that is. A check in the build catches a new entry when Gridline is rebuilt; a check at install catches one that appeared since. Say which this is rather than implying both

## 4. The login screen

Execution note: this group needs the VM at the greeter, and what the login screen lists is the measurement.

- [ ] 4.1 Verify the login screen offers Gridline and the stock Ubuntu session, and not the compositor's entry, reading it off the screen rather than from the filesystem
- [ ] 4.2 Verify the entry the baseline withholds is covered too, by making it visible first. Its `TryExec` names a binary the baseline does not install, so install that binary, confirm the entry would otherwise appear, and confirm the diversion keeps it off the screen. Remove the binary afterwards and verify the removal
- [ ] 4.3 Verify the stock Ubuntu session is still selectable and still works, by logging into it, since it is the fallback every recovery path depends on and the mechanism here is capable of hiding it
- [ ] 4.4 Verify the Gridline session still starts, since nothing in this change should touch it and a diversion that caught the wrong file would show up here

## 5. Removal returns the machine

- [ ] 5.1 Remove `gridline-session` on the VM and verify the login screen offers the compositor's entry again, from the greeter rather than from the package database
- [ ] 5.2 Verify the restored file is the one the compositor's package shipped, by comparing it against the package rather than assuming the diversion put back what it took
- [ ] 5.3 Verify a purge behaves the same as a remove for this, since conffiles and diversions are handled at different stages and the two are easy to assume identical
- [ ] 5.4 Verify nothing of the diversion survives: `dpkg-divert --list` clean, and no leftover file at a diverted name

## 6. Coherence

- [ ] 6.1 Verify each scenario of the two modified capabilities is exercised or recorded as unexercised with a reason, counting them from the delta files rather than from the proposal's description of them
- [ ] 6.2 Verify the modified `package-delivery` requirement reads as a bounded exception rather than a loosening, by confirming the drop-in scenario still stands beside it unchanged and that each obligation the new text adds has a scenario
- [ ] 6.3 Verify `docs/package-matrix.md` and `docs/security-and-manageability.md` describe what was built. Both already record this decision, so the work is confirming they match the implementation rather than writing them, and correcting either if it does not
- [ ] 6.4 Update `docs/recovery.md`, whose administrator recipe now runs in the opposite direction: it documented hiding an entry Gridline does not hide, and the useful recipe is restoring one Gridline diverted. Verify by grep that no live document still describes Gridline as hiding no session entry

## Workflow follow-up

- Run `/opsx:verify` once the login screen and the removal have both been exercised on the VM.
- Archive, then `add-desktop-shell-components`, whose scenario about no other session starting its components this change partly satisfies: the accidental path is gone, and the underlying configuration leak stays recorded and owned where it is.
