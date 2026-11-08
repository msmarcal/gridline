# Proposal

## Why

A Gridline session reaches a desktop with a bar, notifications and a background, and no way to start a program except opening a terminal and typing its name. Nothing in the session presents the applications installed on the machine.

The session's own actions are in the same state. Logging out is `SUPER SHIFT E` and locking is `SUPER Escape`, and both are discoverable only by reading the shipped compositor configuration. A user who did not build this cannot find either.

Five changes so far made the session honest about its own failures. This is the first one about it being usable: a desktop nobody can open a program from is not a desktop.

## What Changes

**This change ships implementation.**

- An application launcher, on a keybinding, that finds and starts installed applications without a terminal.
- A command menu for the session's own actions - lock, log out, suspend, restart, power off - so that each is reachable without knowing a key combination or a command.
- Both are on demand rather than long running, which the session's record and the diagnostic already have a word for: the screen locker is the same shape.
- Neither is blocking. A session whose launcher is not installed still starts, and the gap is reported.
- The compositor binding names a Gridline program rather than the launcher package, which is the rule `defaults.conf` already states about itself: nothing in it names a component Gridline could swap.

**A word about the word "launcher".** In this project it already means `gridline-session`, the program the display manager starts, and `desktop-session` carries three requirements about it. The two things here are therefore always the **application launcher** and the **command menu**, never a bare "launcher". The collision is worth naming rather than discovering in a grep six months from now.

**Out of scope.**

- Choosing the launcher at runtime, or supporting more than one. The project's pattern for a swappable component is a Gridline program that decides, and that program exists here with one answer in it.
- Searching anything other than installed applications: no files, no web, no calculator, no running-window switcher.
- Theming beyond what the session already ships for the bar. The launcher gets a shipped default a user can decline, on the same terms as every other component's configuration.
- Making the command menu extensible by the user or an administrator. The actions in it are the session's own, fixed in this change.
- The `shutdown` capability the adapter already advertises without a verb to perform it. This change needs a way to end a session and will add one; making the adapter's capability list and its verb list agree in general is a different problem.

## Capabilities

### New Capabilities

None. `desktop-session` is the capability that owns what a session presents, and these are two more things it presents.

### Modified Capabilities

- `desktop-session`: two requirements are **added**. One for starting an application without a terminal, one for reaching the session's own actions without knowing their commands. Nothing existing is modified: "The session presents a desktop rather than a bare compositor" enumerates three long-running components and gives them restart semantics that are wrong for a program which runs on demand and exits, so the launcher does not belong inside it.

## Impact

- `packages/gridline/src/adapters/hyprland/defaults.conf`: two keybindings. `SUPER D` and `SUPER SHIFT D` are both free, verified against every bind in the file.
- `packages/gridline/src/adapters/hyprland/adapter`: a verb to end the session, because the compositor's own `exit` keyword is this compositor's vocabulary and `compositor-abstraction` confines that to the adapter.
- `packages/gridline/src/session/`: the Gridline programs the bindings name, and the command menu's actions.
- `packages/gridline/src/config/`: the launcher's shipped configuration.
- `packages/gridline/debian/control`: a dependency on `fuzzel`, which is in the Ubuntu archive at 1.12.0. `package-delivery` already requires dependencies to be declared and satisfiable from the supported archive, so no spec change follows from it.
- `packages/gridline/src/tools/gridline` and the component record, if the launcher is a component the diagnostic reports.
- `docs/`: `component-ownership.md` gains two rows, and `user-experience.md` names both bindings.

## Prior measurement

`fuzzel` was chosen over `tofi`, `wofi` and `bemenu`, all four of which are in the archive, so the project's preference for archive packages did not settle it. It is Wayland-native through layer-shell rather than through XWayland, it is maintained, and its `--dmenu` mode means the command menu is the same binary rather than a second dependency. `wofi` is unmaintained since 2022 and pulls GTK into the launcher path; `tofi` ships no application icons.

**What that mode does was then measured, and the obvious reading of it is wrong.** Plain `--dmenu` returns whatever the user typed when it matches no entry, with a success status, so a menu dispatching on the returned text would perform an action a user invented by mistyping - and two of the five actions here lose work. The command menu therefore dispatches on `--index`, where a mistyped entry comes back as `-1`. design.md carries the decision; it is noted here because the sentence above would otherwise read as though choosing the package settled the mechanism, and it did not.
