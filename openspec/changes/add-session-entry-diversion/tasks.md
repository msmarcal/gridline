# Tasks

## 1. The measurements the design left open

- [x] 1.1 The shape is `dpkg-divert --package gridline-session --divert <path><suffix> --rename --add <path>`. `--package` attributes the diversion to the package that needs it, which is what a package does; `--local` exists and belongs to the administrator. Confirmed by `dpkg-divert --list` and by the login screen offering Gridline and Ubuntu only.

  Two things this settled that the task did not ask. The diverted file stays in the same directory under a different name, and the display manager still ignores it, because it filters by the `.desktop` extension: measured by looking at the selector with both files renamed in place. And `dpkg --verify hyprland` stays clean, which is the difference between diverting and mutilating, since dpkg knows the file moved rather than finding the package corrupt
- [x] 1.2 **Neither candidate this task offered was right.** A check against the archive's file list reports what the compositor shipped when Gridline was built; an enumeration in the installation script runs at install time. The event that breaks the diversion is the compositor's package gaining an entry in a later version, on the user's machine, after both of those moments have passed.

  A dpkg trigger on the session entry directory answers it instead: the installation script is re-run whenever any package changes that directory, so the enumeration happens again when it matters. This is ordinary on the baseline rather than inventive, with eight path triggers already registered and four of them on directories under `/usr/share`. Recorded in design.md, which replaces the coverage-check decision this task was written against
- [x] 1.3 `dpkg -L hyprland` lists exactly two files under the session entry directory, `hyprland.desktop` and `hyprland-uwsm.desktop`, and no third. The implementation does not hard-code those two: it enumerates the package at run time, which is what lets the trigger cover an entry added later

## 2. The diversion

- [x] 2.1 Both registered by the package's own `postinst`, not by hand: `dpkg-divert --list` names each as diverted by `gridline-session`, and the directory holds `gridline.desktop`, `ubuntu.desktop` and the two renamed files
- [ ] 2.2 Add a `postrm` removing every diversion the package registered. Verify by removing the package and confirming `dpkg-divert --list` is clean and each file is back at its original path
- [x] 2.3 `dpkg --verify hyprland` reports nothing, with the diversion in place and again after the package was reinstalled over it
- [x] 2.4 Reinstalled the compositor with the diversion in place. Both diversions survived, the package's files landed at the diverted paths rather than the original ones, and the session entry directory still offers only `gridline.desktop` and `ubuntu.desktop`. This is the behaviour that makes a diversion the right mechanism and an edit the wrong one
- [x] 2.5 Both gates pass and the ownership gate reports 15 files, unchanged, since maintainer scripts and the trigger are package metadata rather than shipped paths

## 3. Coverage by construction

- [x] 3.1 Registered, and confirmed as `/usr/share/wayland-sessions gridline-session/noawait` in dpkg's trigger table. Verified by the mechanism rather than by the declaration: a throwaway package built for the purpose was installed with a file in that directory, and dpkg reported `Processing triggers for gridline-session`. The probe was purged afterwards and its removal verified.

  The probe's own entry was **not** diverted, which is correct and worth recording: the script diverts what the compositor's package owns, not whatever appears in the directory
- [x] 3.2 Simulated by adding a third entry to the compositor package's own file list and creating the file, so `dpkg -L` reported three. Firing the trigger diverted the new one, leaving three diversions, with Gridline neither reinstalled nor reconfigured. The file list was restored from a backup afterwards and the restoration verified by re-counting.

  A near miss worth recording: `dpkg -L` interleaves `diverted by ... to: ...` lines with the paths. The script's pattern is anchored at both ends, so those lines are excluded twice over, by the leading `/` and by requiring the line to end in `.desktop`. A looser pattern would have tried to divert the already-diverted paths
- [x] 3.3 Fired three further times after installation. No error on any run, the diversion set stayed at two, and nothing doubled in the directory. The guard is a `dpkg-divert --list` check before adding, because re-adding an existing diversion is an error rather than a no-op

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
