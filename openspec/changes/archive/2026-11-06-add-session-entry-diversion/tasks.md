# Tasks

## 1. The measurements the design left open

- [x] 1.1 The shape is `dpkg-divert --package gridline-session --divert <path><suffix> --rename --add <path>`. `--package` attributes the diversion to the package that needs it, which is what a package does; `--local` exists and belongs to the administrator. Confirmed by `dpkg-divert --list` and by the login screen offering Gridline and Ubuntu only.

  Two things this settled that the task did not ask. The diverted file stays in the same directory under a different name, and the display manager still ignores it, because it filters by the `.desktop` extension: measured by looking at the selector with both files renamed in place. And `dpkg --verify hyprland` stays clean, which is the difference between diverting and mutilating, since dpkg knows the file moved rather than finding the package corrupt
- [x] 1.2 **Neither candidate this task offered was right.** A check against the archive's file list reports what the compositor shipped when Gridline was built; an enumeration in the installation script runs at install time. The event that breaks the diversion is the compositor's package gaining an entry in a later version, on the user's machine, after both of those moments have passed.

  A dpkg trigger on the session entry directory answers it instead: the installation script is re-run whenever any package changes that directory, so the enumeration happens again when it matters. This is ordinary on the baseline rather than inventive, with eight path triggers already registered and four of them on directories under `/usr/share`. Recorded in design.md, which replaces the coverage-check decision this task was written against
- [x] 1.3 `dpkg -L hyprland` lists exactly two files under the session entry directory, `hyprland.desktop` and `hyprland-uwsm.desktop`, and no third. The implementation does not hard-code those two: it enumerates the package at run time, which is what lets the trigger cover an entry added later

## 2. The diversion

- [x] 2.1 Both registered by the package's own `postinst`, not by hand: `dpkg-divert --list` names each as diverted by `gridline-session`, and the directory holds `gridline.desktop`, `ubuntu.desktop` and the two renamed files
- [x] 2.2 Written, and exercised twice rather than once: on `apt-get remove` and again on `apt-get purge`, each leaving `dpkg-divert --list` empty for this package's suffix and both entries back at their original paths. It reads what to undo from `dpkg-divert` rather than from a list this package wrote, so it removes what was added rather than what someone remembered to record
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

- [x] 4.1 Two entries on the greeter, Gridline and Ubuntu, read off the screen by the person looking at it. The compositor's entry is not offered
- [x] 4.2 Made visible before being tested, which is the whole point: `hyprland-uwsm.desktop` carries `TryExec=uwsm`, and with `uwsm` absent the greeter withholds it whether or not Gridline exists. Installing `uwsm` removes that excuse. With it present the greeter still listed two entries, confirmed on screen, so the second entry is covered by the diversion and not by the baseline's omission.

  Without this step the task would have confirmed that something already invisible was invisible
- [x] 4.3 Selected from the greeter and logged into, and it works. This is the one the mechanism must never be pointed at, so exercising it rather than reasoning from the diversion list is the point
- [x] 4.4 Selected and logged into, and it works. A diversion that had caught `gridline.desktop` would have removed the entry entirely, so this is the cheapest check that the right files were taken

## 5. Removal returns the machine

- [x] 5.1 Removed on the VM, and the greeter was read by the person looking at it: Hyprland and Ubuntu, with Gridline gone because its package went and the compositor's entry back because the diversion was undone. The uwsm variant was withheld again by its own `TryExec`, the borrowed `uwsm` having been returned first, which is the baseline's behaviour rather than ours.

  The system side agreed before the screen was looked at, with zero diversions and `dpkg --verify hyprland` clean, and the screen is what the delta promises: a machine without Gridline offers what it offered before Gridline arrived
- [x] 5.2 `dpkg --verify hyprland` reports nothing after the removal, which compares both restored files against the package's own checksums rather than against a copy this change kept. That is the comparison the task asks for and it is stronger than one against a copy, since a copy could be wrong in the same way twice
- [x] 5.3 Exercised separately rather than assumed. After `apt-get purge`, zero diversions remain, both entries are back at their original paths, and `dpkg --verify hyprland` is clean, the same end state as `apt-get remove`
- [x] 5.4 After both the remove and the purge, `dpkg-divert --list` filtered to this package's suffix returns nothing, and the session entry directory holds no file carrying the diverted name

## 6. Coherence

- [x] 6.1 Eleven scenarios, counted from the delta files: four under `desktop-session`, seven under `package-delivery`. All accounted for.

  **`desktop-session`.** *One tiling session is offered*: 4.1, read off the greeter. *The compositor's entry returns when Gridline goes*: 5.1, read off the greeter after removal, with the system side agreeing first. *The fallback session is never hidden*: 4.3, the stock session selected and used; the mechanism is capable of hiding it and is not pointed at it. *The compositor stays usable to someone who wants it*: the compositor remains installed and startable, and `docs/recovery.md` now documents restoring its entry; the restoration recipe itself is **not** exercised and is recorded here as such rather than counted.

  **`package-delivery`.** The three scenarios this change did not touch are unaffected and still hold: every declared path is Gridline-named, each has one owner, and the build reports a manifest declaring an archive path. *Influence on a third-party component uses a drop-in* is unchanged and still stands, which is the point of the modification being bounded. *A diversion is used only where a drop-in cannot reach*: the reason is recorded in `docs/package-matrix.md` where a reader meets the diversion. *A diversion does not outlive the package that needs it*: 2.2, 5.1 through 5.4, on both remove and purge. *A diversion covers every file it has to*: 3.1 through 3.3, where the trigger makes coverage structural rather than checked
- [x] 6.2 The drop-in scenario is unchanged and still present, now reading as the rule for the case a directory can carry the change. The added text attaches three obligations and each has its own scenario: used only where a drop-in cannot reach, does not outlive the package, covers every file it has to. The requirement is longer and narrower rather than weaker, and a reader meeting it finds the exception fenced rather than the rule relaxed
- [x] 6.3 They recorded the decision and predated the mechanism, so both needed correcting rather than confirming. `docs/package-matrix.md` described a single entry where the compositor ships two, and warned that covering one and not the other would silently stop working, which the trigger now prevents rather than merely warns about. It now describes the run-time enumeration, the trigger, and the removal reading its own diversions back from dpkg. `docs/security-and-manageability.md` records the accepted cost and needed no change, since nothing about the cost moved
- [x] 6.4 Rewritten. It said **"Gridline hides none of them, and will not"**, which this change makes false, and its recipe hid an entry Gridline now hides by itself. The useful recipe is the opposite one, restoring the compositor's entry without removing Gridline, and it is there with the warning that the restored session is not a Gridline session: it reads Gridline's compositor defaults and has none of Gridline's components, so its lock key does nothing.

  The hiding recipe is kept, since the mechanism is general, along with the warning never to point it at the stock Ubuntu session. Verified by grep that no live document still says Gridline hides nothing

## 7. The compositor's name leaves the maintainer script

Added by `/opsx:verify`, which found the `postinst` naming the compositor package directly. `compositor-abstraction` requires that components outside the adapter layer contain no compositor-specific logic, and the build gate could not see this one: it excludes `debian/` wholesale, on a reason written into the gate itself that covers declarative metadata and says in so many words that it is not covering logic that leaked out of the adapter. A `Depends` on a compositor is a recorded dependency; a script that enumerates a named package and acts on the result is not.

- [x] 7.1 Declare the compositor package inside the adapter, at `src/adapters/<compositor>/compositor-package`, following the shape `portal-backend` already set for the portal mapping. Verify the isolation check counts it as adapter content rather than a leak
- [x] 7.2 Turn the maintainer script into a template carrying no compositor name, and substitute at build time with `tools/generate-maintainer-scripts`, which fails rather than ships if no adapter declares a package or if the placeholder survives. Verified the built package's `postinst` carries the substituted name, read with `dpkg-deb -I`.

  **The output is a generated file at a tracked path**, which `git status` caught and the precedent did not have to solve: `generate-portals-conf` writes into `debian/generated/`, which is already ignored, while `dh_installdeb` fixes the name a maintainer script must have. A `.gitignore` entry now covers it, with the reason, and the previously committed copy has to be untracked once by hand, since an ignore rule does not undo tracking. Without both, every build leaves the tree dirty with an artefact
- [x] 7.3 Verify the gate now catches what it could not before, which is the whole point of moving the file rather than only rewriting it. The template lives under `src/config/`, which the check scans: planting the compositor's name back into it fails the build with `src/config/gridline-session.postinst.in references hyprland`, and removing it passes. Before this change that same name sat in `debian/` and no gate had anything to say
- [x] 7.4 Verify the behaviour is unchanged by the refactor, by removing the package and installing it again so the generated script does the work from nothing: two diversions registered, the session entry directory holding only Gridline's and Ubuntu's entries, `dpkg --verify hyprland` clean, and the trigger registered

## Workflow follow-up

- Run `/opsx:verify` once the login screen and the removal have both been exercised on the VM.
- Archive, then `add-desktop-shell-components`, whose scenario about no other session starting its components this change partly satisfies: the accidental path is gone, and the underlying configuration leak stays recorded and owned where it is.
