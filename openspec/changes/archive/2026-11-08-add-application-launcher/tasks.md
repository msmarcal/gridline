# Tasks

## 1. What the baseline does before anything is added

The three components before this one were each measured before being configured, and each time the measurement changed what got built. The wallpaper daemon's published configuration syntax did not apply to the shipped version, and the bar's configuration was unreachable on every machine because of a search path nobody had checked.

- [x] 1.1 Install `fuzzel` on the VM and run it with no configuration at all, in a Gridline session. Record what it shows, whether it finds installed applications, and what it looks like, because that is the fallback a user gets when Gridline's configuration is declined
- [x] 1.2 Establish which configuration paths the shipped version actually searches, and in what order, from the version on the VM rather than from documentation. The bar's defect was exactly this: a system path the component's own package already occupied
- [x] 1.3 Establish whether the shipped version accepts the configuration keys this change intends to set, by setting them and observing a change. Unknown keys are accepted silently by more than one component here
- [x] 1.4 Run `fuzzel --dmenu` with a list on standard input and record how a selection is returned, and what is returned when the user cancels instead of choosing. The command menu's correctness rests on telling those two apart

## 2. The application launcher

- [x] 2.1 Add the Gridline program the binding names, which starts the launcher and decides whether to name Gridline's configuration, following `gridline-shell`'s existing shape. Verify it execs the launcher with no arguments outside a Gridline session, by running it with `XDG_CURRENT_DESKTOP` unset
- [x] 2.2 Ship the launcher's configuration, using only keys proven in 1.3 to have an effect. Verify the running launcher reflects each key by observing the change, not by the file being present
- [x] 2.3 Add the `SUPER D` binding to `defaults.conf`, naming the Gridline program and not the launcher. Verify from a running session that the key opens it, and verify by reading the file that no package name appears in the binding
- [x] 2.4 Declare the dependency in `debian/control` and verify the built package's declared dependency is satisfiable from the archive with `apt-get install -s`
- [x] 2.5 Verify a user's own configuration wins: place one, confirm it applies and that Gridline's is not reimposed, which is what `desktop-session` requires of every component's configuration

## 3. The command menu

- [x] 3.1 Add the adapter verb that ends the session, named for the capability the adapter already advertises rather than for the compositor's keyword. Verify the build gate that keeps compositor vocabulary inside the adapter still passes, and verify the verb from outside the adapter by calling it
- [x] 3.2 Add the Gridline program that prints the action list and performs the chosen action. Verify each of the five actions separately, and verify that cancelling performs nothing, which 1.4 establishes how to detect
- [x] 3.3 Verify locking from the menu takes the same path as `SUPER Escape` and the idle timeout, by observing that the lock unit is what starts, rather than a locker started by the menu
- [x] 3.4 Verify ending the session releases the seat and returns to the display manager, which `desktop-session` already requires of session teardown however it was triggered
- [x] 3.5 Verify the adapter-unreachable case: the menu reports the failure and does not fall back to ending the session another way. Measure it by making the adapter unreachable rather than by reading the code
- [x] 3.6 Add the `SUPER SHIFT D` binding and verify it from a running session

## 4. Absence, which is the case a user meets by pressing a key

- [x] 4.1 Verify a session starts and reaches a usable desktop with the launcher not installed, and that the remaining components are unaffected
- [x] 4.2 Verify the diagnostic names the launcher as absent when it is not installed, under the project's word for it rather than the package's name
- [x] 4.3 Verify the diagnostic does not report the launcher as stopped or failed during a healthy session, which is the `ready` state's whole purpose and the thing the previous change got wrong for the screen locker
- [x] 4.4 Verify every action the command menu offers remains available without it, by the key combination or command that existed before

## 5. Nothing leaks into other sessions

- [x] 5.1 Verify a session that is not Gridline's is unchanged: the bindings are in Gridline's own compositor configuration and reach nothing else, and the Gridline programs name no configuration outside a Gridline session. Measure from a session that is not Gridline's rather than by reasoning about the files
- [x] 5.2 Verify the build gates still pass, including the one that checks a Gridline package declares only Gridline-named paths

## 6. Coherence

- [x] 6.1 Add the two rows to `docs/component-ownership.md`, stating which failure each produces, and name both bindings in `docs/user-experience.md`
- [x] 6.2 Verify nothing in the repository now uses a bare "launcher" to mean the application launcher, which in this project already means `gridline-session`. Grep rather than recall
- [x] 6.3 Verify each scenario of the two added requirements is exercised or recorded as unexercised with a reason, counting scenarios from the delta file rather than from the proposal's description of them

## Scenario accounting

Thirteen scenarios across the two added requirements, counted from
`specs/desktop-session/spec.md` rather than from the proposal's description of
them. All thirteen were exercised on the LXD VM. Where a person pressed the
key, that is said, because this project's test harness cannot: the compositor
does not act on its own key bindings when they come from a virtual keyboard,
measured by sending `SUPER Return` - a binding that has worked since the first
day - and watching nothing happen.

**An application can be started without a terminal**

1. **An installed application can be found and started** - typed `kitty`, chose
   it, and a `kitty` window appeared in the compositor's own client list. The
   first attempt read "it did not start" from `pgrep -x`, which cannot match a
   process name over 15 characters and `gnome-calculator` is 16; the calculator
   had started all along and was sitting in the client list unnoticed.
2. **Reaching it needs no pointer and no prior knowledge of a command** -
   `SUPER D` pressed by a person, opening the launcher. The combination is in
   the shipped compositor configuration, readable with no session running.
3. **Not running is not a fault** - `ready` in a healthy session, never
   `stopped` or `failed`, which is what the previous change got wrong for the
   screen locker.
4. **An absent launcher degrades the desktop and does not block the session** -
   a fresh login with the launcher moved aside reached a usable desktop with
   bar, notifications and authentication agent running. **This scenario found a
   defect while passing**: the diagnostic reported `ready` from the previous
   session's record, because nothing wrote the launcher down at session start.
   Fixed with a second component list in the session launcher, and both states
   measured by running the shipped loop rather than a copy of it.
5. **The session's binding does not name the launcher it ships** - read back
   from the installed configuration: the binding names `gridline-menu apps`.
6. **Its configuration is a shipped default a user can decline** - measured
   from `/proc/<pid>/cmdline`, which is the argument vector as it is rather
   than as the program meant it: Gridline's path named inside a Gridline
   session, nothing named when the user has written their own, and Gridline's
   returning when theirs is removed.
7. **Only the Gridline session is affected** - outside a Gridline session the
   launcher runs bare, the command menu refuses, no record is written at all,
   and no Gridline package owns a file under any of the launcher's own paths.

**The session's own actions are reachable without knowing their commands**

8. **Each action is listed and can be chosen** - all five seen on screen by a
   person, and each dispatched against a stub, because four of them end the
   machine or the session and cannot be performed to find out whether they
   were dispatched correctly.
9. **An action asked for here is the same action asked for elsewhere** -
   end to end, by a person: the menu's lock entry left
   `gridline-lock.service` active with the locker inside that unit's own
   cgroup, which is precisely what a menu running a locker itself would not do.
10. **Ending the session releases the seat** - chosen by a person; logind
    removed the session and the greeter took seat0.
11. **An action that cannot be performed says so** - a refused action raised a
    desktop notification, which the person at the machine then photographed
    without being asked to. stderr carries it too, for the journal.
12. **Losing it costs no action** - every action it offers has a key binding or
    a command that predates it, each checked for presence.
13. **Only the Gridline session is affected** - the command menu refuses
    outside a Gridline session rather than offering a session's actions to a
    session that is not it.

## Findings that are not this change's to fix

- **The compositor segfaults when a session ends on this VM.** Twenty-one
  occurrences in the journal, every one of them `segfault at 10` in
  `libaquamarine`, at the same offset. Two of them line up with session
  teardowns from before this change existed, so the adapter's new verb is not
  the cause: it is the graphics backend on a machine with no 3D. Worth a change
  of its own, and worth knowing before anybody reads a crash report after a
  logout and starts looking in the wrong place.
- **A key binding cannot be exercised by this project's test harness.** The
  compositor does not act on its own bindings from a virtual keyboard client.
  `add-ci-vm-smoke-tests` needs to know this before it plans to drive a desktop
  with `wtype`: every binding it wants to test needs either a person or a
  different mechanism.

## Workflow follow-up

- Run `/opsx:verify` once the components are in place and absence has been measured.
- The unconfirmed command menu is a decision to revisit with evidence. If somebody loses work to a keystroke, that is the measurement that reopens it, and it belongs to its own change rather than to this one.
