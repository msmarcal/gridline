# Tasks

## 1. What the baseline does before anything is added

The three components before this one were each measured before being configured, and each time the measurement changed what got built. The wallpaper daemon's published configuration syntax did not apply to the shipped version, and the bar's configuration was unreachable on every machine because of a search path nobody had checked.

- [ ] 1.1 Install `fuzzel` on the VM and run it with no configuration at all, in a Gridline session. Record what it shows, whether it finds installed applications, and what it looks like, because that is the fallback a user gets when Gridline's configuration is declined
- [ ] 1.2 Establish which configuration paths the shipped version actually searches, and in what order, from the version on the VM rather than from documentation. The bar's defect was exactly this: a system path the component's own package already occupied
- [ ] 1.3 Establish whether the shipped version accepts the configuration keys this change intends to set, by setting them and observing a change. Unknown keys are accepted silently by more than one component here
- [ ] 1.4 Run `fuzzel --dmenu` with a list on standard input and record how a selection is returned, and what is returned when the user cancels instead of choosing. The command menu's correctness rests on telling those two apart

## 2. The application launcher

- [ ] 2.1 Add the Gridline program the binding names, which starts the launcher and decides whether to name Gridline's configuration, following `gridline-shell`'s existing shape. Verify it execs the launcher with no arguments outside a Gridline session, by running it with `XDG_CURRENT_DESKTOP` unset
- [ ] 2.2 Ship the launcher's configuration, using only keys proven in 1.3 to have an effect. Verify the running launcher reflects each key by observing the change, not by the file being present
- [ ] 2.3 Add the `SUPER D` binding to `defaults.conf`, naming the Gridline program and not the launcher. Verify from a running session that the key opens it, and verify by reading the file that no package name appears in the binding
- [ ] 2.4 Declare the dependency in `debian/control` and verify the built package's declared dependency is satisfiable from the archive with `apt-get install -s`
- [ ] 2.5 Verify a user's own configuration wins: place one, confirm it applies and that Gridline's is not reimposed, which is what `desktop-session` requires of every component's configuration

## 3. The command menu

- [ ] 3.1 Add the adapter verb that ends the session, named for the capability the adapter already advertises rather than for the compositor's keyword. Verify the build gate that keeps compositor vocabulary inside the adapter still passes, and verify the verb from outside the adapter by calling it
- [ ] 3.2 Add the Gridline program that prints the action list and performs the chosen action. Verify each of the five actions separately, and verify that cancelling performs nothing, which 1.4 establishes how to detect
- [ ] 3.3 Verify locking from the menu takes the same path as `SUPER Escape` and the idle timeout, by observing that the lock unit is what starts, rather than a locker started by the menu
- [ ] 3.4 Verify ending the session releases the seat and returns to the display manager, which `desktop-session` already requires of session teardown however it was triggered
- [ ] 3.5 Verify the adapter-unreachable case: the menu reports the failure and does not fall back to ending the session another way. Measure it by making the adapter unreachable rather than by reading the code
- [ ] 3.6 Add the `SUPER SHIFT D` binding and verify it from a running session

## 4. Absence, which is the case a user meets by pressing a key

- [ ] 4.1 Verify a session starts and reaches a usable desktop with the launcher not installed, and that the remaining components are unaffected
- [ ] 4.2 Verify the diagnostic names the launcher as absent when it is not installed, under the project's word for it rather than the package's name
- [ ] 4.3 Verify the diagnostic does not report the launcher as stopped or failed during a healthy session, which is the `ready` state's whole purpose and the thing the previous change got wrong for the screen locker
- [ ] 4.4 Verify every action the command menu offers remains available without it, by the key combination or command that existed before

## 5. Nothing leaks into other sessions

- [ ] 5.1 Verify a session that is not Gridline's is unchanged: the bindings are in Gridline's own compositor configuration and reach nothing else, and the Gridline programs name no configuration outside a Gridline session. Measure from a session that is not Gridline's rather than by reasoning about the files
- [ ] 5.2 Verify the build gates still pass, including the one that checks a Gridline package declares only Gridline-named paths

## 6. Coherence

- [ ] 6.1 Add the two rows to `docs/component-ownership.md`, stating which failure each produces, and name both bindings in `docs/user-experience.md`
- [ ] 6.2 Verify nothing in the repository now uses a bare "launcher" to mean the application launcher, which in this project already means `gridline-session`. Grep rather than recall
- [ ] 6.3 Verify each scenario of the two added requirements is exercised or recorded as unexercised with a reason, counting scenarios from the delta file rather than from the proposal's description of them

## Workflow follow-up

- Run `/opsx:verify` once the components are in place and absence has been measured.
- The unconfirmed command menu is a decision to revisit with evidence. If somebody loses work to a keystroke, that is the measurement that reopens it, and it belongs to its own change rather than to this one.
