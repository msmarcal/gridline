# Tasks

## 1. The relationship between Gridline's own packages

- [x] 1.1 Reproduce the defect first: with both packages installed, remove `gridline-session` on the VM and record what `gridline doctor` then does, so the fix has something to be measured against rather than being verified against its own intention
- [x] 1.2 Declare `gridline-session` in `gridline-tools`. Verify from the built package with `dpkg-deb -f gridline-tools Depends`, and verify the direction is one way only by confirming `gridline-session` gained nothing
- [x] 1.3 Verify removing `gridline-tools` leaves a working session, and that removing `gridline-session` now takes `gridline-tools` with it rather than leaving a command that cannot answer. Measure with `apt-get remove -s` for both directions

## 2. The terminal becomes a preference

These land together or not at all. The list's comment justifies its single entry by the terminal being a dependency, so changing one without the other leaves the file describing a rule it no longer follows.

- [x] 2.1 Establish what the session does today with no terminal installed at all, before anything changes: move `kitty` aside, press the binding, and record whether a terminal opens and which. This is the baseline the fallback has to beat
- [x] 2.2 Change `Depends: kitty` to `Recommends: kitty` and verify from the built package that `kitty` appears in `Recommends` and not in `Depends`
- [x] 2.3 Add the second entry to `gridline-xdg-terminals.list` and rewrite the comment that justifies a single entry, since its reasoning stops being true. Verify the installed file holds both entries in order
- [x] 2.4 Verify removing `kitty` no longer removes `gridline-session`, with `apt-get remove -s kitty`, which today lists the session for removal
- [x] 2.5 Verify Gridline's preference still wins when `kitty` is present: press the binding in a Gridline session and confirm which terminal opens, and confirm the same machine's non-Gridline session is unaffected
- [x] 2.6 Verify the fallback: with `kitty` absent, the binding opens the terminal the second entry names, not an arbitrary one. Compare against what 2.1 recorded
- [x] 2.7 Verify an `--no-install-recommends` install is still a usable session, or record precisely what it loses. Measure with `apt-get install -s --no-install-recommends` against the built package rather than reasoning from the field

## 3. What the package says it assumes

- [x] 3.1 State the display manager assumption in `gridline-session`'s long description: that the session entry is read by one, that Gridline installs none, and that the entry is inert without one. Verify by reading `dpkg-deb -f gridline-session Description` from the built package
- [x] 3.2 Verify the description still matches what the package ships, by comparing it against `dpkg-deb -c`. Three descriptions in this project have claimed "no content yet" while shipping files, so this comparison is the one that catches it
- [x] 3.3 Verify no second place now states the same assumption as a dependency, which the added requirement forbids: grep the control file and the docs for a display manager declaration

## 4. Removal still returns a stock desktop

- [x] 4.1 Verify removing all three Gridline packages with `autoremove` leaves `ubuntu-desktop`, `gnome-shell`, `gdm3` and `mutter` installed, after the dependency changes rather than before. The earlier measurement was taken with the old dependencies and does not carry over
- [x] 4.2 Verify the measurement is not reading the test machine's history: check whether each package involved is marked manual or auto before quoting the result, because the first attempt at this reported five packages surviving for no reason other than having been installed by hand

## 5. Coherence

- [x] 5.1 Correct `docs/package-matrix.md` where it describes the terminal preference, which states that the terminal named there is a dependency of `gridline-session` and explains the single entry by it
- [x] 5.2 Correct `docs/component-ownership.md`'s terminal row if it states the dependency relationship, found by grep rather than by recall
- [x] 5.3 Verify each scenario of the two added requirements is exercised or recorded as unexercised with a reason, counting scenarios from the delta file rather than from the proposal's description of them

## Scenario accounting

Seven scenarios across the two added requirements, counted from
`specs/package-delivery/spec.md`. Six were exercised; one is recorded as
unexercised with its reason rather than counted as passing.

**A package states what it assumes and does not declare**

1. **The session says what it needs a machine to already have** - the paragraph
   is in `gridline-session`'s long description and was read back from the built
   package with `dpkg-deb -f`, not from the source file.
2. **What the package ships is described, not promised** - all three packages
   compared against `dpkg-deb -c`: 16, 9 and 1 files, none claiming to ship
   none. The same comparison also checked each component the description names
   against the field that declares it. It found the description listing three
   non-blocking components when there are four, and `libnotify-bin` missing
   from the sentence about which dependencies are in main; both corrected.
3. **An assumption that becomes a dependency stops being an assumption** -
   **unexercised.** Nothing in this change makes that transition: the terminal
   moved from a dependency to a recommendation, which is the other direction,
   and the display manager assumption is new. The scenario is a rule for the
   change that declares a display manager one day, and exercising it would mean
   declaring one in order to test the rule against it.

**A component reached through a mechanism is replaceable without removing Gridline**

4. **The preferred component can be removed on its own** - `apt-get remove -s
   kitty` lists only `kitty`. Before this change it listed `gridline-session`
   with it.
5. **The preference still applies when nothing else expresses one** - with the
   terminal installed, the session opened it, read from the compositor's own
   client list rather than from the command.
6. **Removing the preferred component degrades to the mechanism's answer** -
   and this one nearly passed for the wrong reason. With the terminal removed
   from a machine carrying two, the list and the scan give the same answer, so
   the measurement could not tell which had answered. A third terminal was
   installed to make them disagree. The scan then picked one that needs X11 and
   the session opened nothing at all; with Gridline's second entry present the
   same removal opened the terminal Gridline names.
7. **A component Gridline configures by name stays a dependency** - the bar,
   the notification daemon, the wallpaper daemon and the application launcher
   are each still in `Depends`, checked from the built package against the
   description that names them.

## Findings

- **An unrelated package can break a keybinding, and nothing says so.**
  Installing `xterm` changed which terminal a Gridline session opens, from a
  working one to one that cannot run under Wayland, and the failure is a key
  that does nothing. The second list entry fixes it for the terminal. Nothing
  in the session notices or reports it, and whether that is worth reporting is
  not this change's question.

## Workflow follow-up

- Run `/opsx:verify` once the terminal change and the declaration are both in and both measured.
- The pre-existing `--strict` failure in `package-delivery`, a requirement whose prose runs to 713 characters, is not this change's to fix. It predates it and belongs to a change of its own rather than being folded in here.
