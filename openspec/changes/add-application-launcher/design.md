# Design

## Context

See proposal.md for motivation. What the session already has, because all three constraints below come from existing files rather than from preference.

**The compositor configuration states a rule about itself.** `defaults.conf` says, in its own header, that nothing in it names a component Gridline could swap, and it keeps that rule twice: the terminal binding calls `xdg-terminal-exec`, the freedesktop mechanism for whichever terminal the system is configured to use, and the lock binding calls `loginctl lock-session` rather than a locker. A binding that said `fuzzel` would be the first exception, and would put the choice of launcher in a file users are expected to copy and edit.

**There is already a pattern for a component Gridline starts.** The three desktop shell components start through `gridline-shell`, a program that decides at start time whether to name Gridline's configuration, so the same drop-in works in a session that is not Gridline's. The record and the diagnostic read from Gridline's own names rather than from unit internals.

**There is already a word for a component that runs on demand.** `report-components-that-stop` added `ready` for exactly this: installed, able to run, not running because nothing is asking. It was added for the screen locker, and the application launcher is the second thing with that lifetime. Nothing new is needed to describe it.

**The adapter advertises a capability it cannot perform.** `confirm` reports `shutdown` unconditionally, and the adapter has no verb that ends a session. Today ending a session is the compositor's own `exit` keyword, written directly in `defaults.conf`. That is correct placement - compositor vocabulary belongs in the adapter path - but it means the command menu has nothing compositor-neutral to call.

## Goals / Non-Goals

**Goals:**

- A user can start an application and perform the session's actions without knowing a command.
- Replacing the launcher is a change to one Gridline file, not to anybody's keybindings.
- Both components fail the way every other non-blocking component here fails: the session starts, the gap is reported.

**Non-Goals:**

- Choosing the launcher at runtime. One answer, in one place, changeable by editing that place.
- Making the command menu's action list extensible.
- Reconciling the adapter's capability list with its verb list in general. This change adds the one verb it needs.

## Decisions

### Two Gridline programs, named by the bindings, the way the shell components are

`SUPER D` and `SUPER SHIFT D` name Gridline-owned programs, and those programs name the launcher. This is the third use of a shape already in the tree, and it buys the same two things it bought for the bar: the binding keeps working when the launcher changes, and the program is where the decision about Gridline's configuration lives, so it can be absent outside a Gridline session.

The alternative was binding `fuzzel` directly, which is one fewer file. It breaks the rule `defaults.conf` states about itself, and it puts a package name in the file this project most expects users to copy.

**One key apart, deliberately.** The two are the same letter and one modifier, because they are the same gesture - ask the session for something - and a user who learns one has learned the other.

### `--dmenu` is why this is one dependency and not two

The command menu is the same binary in a different mode, reading its entries on standard input. So the actions are a list a Gridline program prints, and the menu is a way of choosing from it. That keeps the action list in a shell program where it can be read, and means the launcher's configuration and appearance apply to both without being described twice.

**And it dispatches on the index, not on the text.** Measured, because the obvious reading of dmenu mode is wrong in a way that matters: plain `--dmenu` returns *whatever the user typed* when it matches no entry, with exit 0. A menu that acted on that string would try to perform an action the user invented by mistyping, and the five actions here include powering off.

`--index` closes it. A chosen entry returns its position; a cancellation returns nothing with exit 2; **text matching nothing returns `-1` with exit 0**, which is a sentinel a program can test rather than a string it has to recognise. So the menu performs an action only for an index within the list it printed, and `-1`, exit 2 and anything else do nothing at all.

This also frees the labels. Because dispatch is positional, what the user reads can be plain words - `Power off`, not a command - which is what the spec asks for and would otherwise have been in tension with matching the returned text.

### Measured about the launcher before anything was configured

Four facts that shaped the above, each from the shipped 1.12.0 rather than from documentation:

- **Its own package ships `/etc/xdg/fuzzel/fuzzel.ini`.** Gridline cannot put its configuration there without overwriting a file another package owns, which the project forbids. This is the bar's defect exactly - waybar ships `/etc/xdg/waybar/config.jsonc` and made Gridline's unreachable on every machine - and it is why the Gridline program names the path with `--config` instead.
- **It reads its configuration before connecting to Wayland**, so configuration can be checked without a session.
- **It rejects an unknown key, an unknown section and an out-of-range value**, naming the file and line. That is the opposite of the wallpaper daemon, which accepts unknown keys in silence. `--check-config` exits 0 or 1 on exactly this question, so the launcher's shipped configuration is machine-checkable. No gate is built here because no task asks for one and the build machine has no fuzzel, but the option exists and the flag is the right instrument.
- **The archive's default is four non-comment lines, all section headers**, so declining Gridline's configuration gives a user plain fuzzel rather than something half-configured. Its appearance is light and does not match the session's bar, which is work the shipped configuration has to do rather than a preference.

### Each action takes the path that already exists, and that is the whole point

Lock asks logind, the same as `SUPER Escape` and the same as the idle timeout, because `session-manageability` requires lock policy to be one policy. Suspend, restart and power off ask logind too. Ending the session asks the adapter.

**The adapter gets a verb, and this is the only new mechanism here.** The compositor's `exit` keyword is this compositor's word, and `compositor-abstraction` confines that vocabulary to the adapter, so the command menu cannot call it. The verb is named for the Gridline capability the adapter already advertises rather than for the compositor's keyword. The alternative, `loginctl terminate-session`, is compositor-neutral and worse: it ends the session from outside rather than asking the compositor to leave, which is the harder of the two and bypasses whatever the compositor does on its way out.

### The launcher is a component the diagnostic knows about

It gets a record under its Gridline name, like the other five, written by the program the binding names. Its usual state is `ready`, and `absent` when the launcher is not installed - which is the answer a user needs, because this is the one component whose absence they will notice by pressing a key and getting nothing.

What it does not get is a systemd unit. The three shell components have units because they are meant to be running; this one is started by a keypress and exits, so a unit would be a wrapper around a thing that is already a program. That means no `ExecStopPost`, so nothing records its end - and nothing should: an application launcher that has exited is `ready`, not `stopped`, and the program that starts it can say so on the way out.

## Risks / Trade-offs

- **`SUPER D` may already mean something to a user coming from another desktop.** → It is the most common binding for this across tiling desktops, and `defaults.conf` is a shipped default a user can decline, which `desktop-session` already requires of every component's configuration.
- **The command menu's actions are destructive and a menu is easier to hit by accident than a key combination.** → Open question below. The spec requires that a refused action says so; it does not yet require confirmation, and whether it should is not decided.
- **A fourth non-blocking component is a fourth thing that can be absent.** → It is reported the way the other three are, and unlike them its absence is immediately visible to the user who pressed the key, which is the easier failure to diagnose rather than the harder one.
- **The adapter gains a verb, so every future adapter has to implement it.** → It is a capability the adapter already claims to provide. The gap was that nothing could use the claim.

### No confirmation, and the reason is that confirmation is cheap to add later and expensive to unlearn

Choosing an action performs it. Nothing in the menu asks twice.

The case for confirming is real: `Power off` sits two rows from `Lock`, and a mistyped letter ends somebody's work. The case against it won: a confirmation a user always accepts teaches them to accept without reading, which removes the protection while keeping the cost, and logind already refuses to power off when another session would lose work. So the unconfirmed version ships, and if it hurts in use, the change that adds confirmation will have a measured reason rather than an argued one.

**This is a decision to revisit with evidence, not a closed one.** What would reopen it is somebody losing work to a keystroke, which is exactly the kind of thing this project has been recording rather than guessing at.

### When the adapter cannot be reached, the menu says so and does nothing else

Ending the session is the one action that is not logind's, so it is the one that can fail in a way the others cannot. It reports the failure and stops there. It does not fall back to `loginctl terminate-session`: a fallback that ends the session a harder way, silently, when the gentle way was unavailable, is the kind of help that loses work. The spec already requires that an action which cannot be performed says so.

## Open Questions

- **Whether the launcher should report `absent` through the same record the launcher-binding program writes, or whether a component nothing supervises deserves its own answer.** The record was built for components a session starts and keeps; this is the first one a keypress starts.
