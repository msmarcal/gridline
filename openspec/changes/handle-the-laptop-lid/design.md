# Design

## Context

See proposal.md for the measurements. What shapes the design is where the project keeps compositor knowledge, what the compositor already does on its own, and that the one machine that exposed this is not a machine the verification VM can imitate.

**Everything compositor-specific stays in the adapter directory.** `compositor-abstraction` requires it and a build check enforces it. Enabling or disabling a named output is a compositor action, so it is a verb of the adapter. Deciding *when* to do it is not, so that is a program outside it, and it can be read and tested without knowing which compositor runs.

**The compositor tells us the lid's state, including at startup.** The log of the owner's session records a `Lid Switch` device and, once, at startup, `Switch Lid Switch turn on, triggering binds` with the lid closed. A binding on the switch therefore runs when the session starts and on every change after, and carries the state in which of the two it is. Nothing has to ask the lid separately, and a session that starts with the lid closed is handled by the same code path as one that closes it later.

**Output names are the kernel's connector names.** The compositor calls the panel `eDP-1` and the monitor `HDMI-A-2`, and `/sys/class/drm/card1-eDP-1` and `card1-HDMI-A-2` carry the same names. That is how the program can tell a built-in panel from an external screen without asking the compositor, which keeps the decision outside the adapter.

**The launcher's configuration displaces nothing.** Gridline's compositor defaults are named by the adapter only when the user has no compositor configuration of their own, and a user who writes one replaces them whole. So these bindings reach a Gridline session and no other, and a user who has written their own configuration has chosen to handle the lid themselves.

## Goals / Non-Goals

**Goals:**

- A session on a lid-closed laptop with an external screen is usable on that screen, from the first frame.
- The panel comes back when the lid opens.
- No path by which this change leaves the session without any output.

**Non-Goals:**

- Layout, primary screen, scaling or any other display policy.
- Hotplug while the lid is closed.
- Putting the panel to sleep for power. The point is that it is not a screen of the session, not that it saves energy.

## Decisions

### A neutral program decides, the adapter acts

`gridline-lid`, installed beside the other session programs, takes one argument, `closed` or `open`, and does one of two things.

- **`closed` and an external screen is connected:** ask the adapter to disable every built-in panel.
- **Anything else:** ask the adapter to enable every built-in panel.

A built-in panel is a connector whose name starts with `eDP`, `LVDS` or `DSI`, which are the kernel's names for laptop panels. An external screen is any other connector whose `status` file reads `connected`. Both come from sysfs, so a tree can be substituted for a test through an environment variable, which is the only reason that variable exists.

The adapter's new verb is `output <name> on|off`. Naming the output and the state is the whole contract; how a compositor turns an output off is its own business, and the verb is within the `output-management` capability the adapter already advertises, so no capability is added and the session's start-up gate is unchanged.

**Enabling is the default branch on purpose.** Every doubt (an unreadable connector tree, an argument that is neither word, a failed adapter call) lands on enable, which is the fail-safe the requirement states: the built-in screen stays in use. The program never disables on anything but a positive reading of both conditions.

### Two bindings on the lid switch, active while locked

```
bindl = , switch:on:Lid Switch,  exec, /usr/libexec/gridline/gridline-lid closed
bindl = , switch:off:Lid Switch, exec, /usr/libexec/gridline/gridline-lid open
```

`bindl` is the flavor that runs while the session is locked, as the media keys already do, so closing the lid at a lock screen is handled and the lock is not touched. These lines are the compositor's vocabulary, which is why they live in the adapter's directory with the rest of the compositor defaults and not in the neutral program.

### Re-enabling uses the output's own preferred mode

The enable action asks for the preferred mode, automatic position and automatic scale. On the owner's panel that produced 1920x1080 at scale 1.5 in the log, which is what a first start produces, so reopening the lid returns the panel to what it was.

## Alternatives considered

- **Leave the panel enabled and place the external screen first.** It avoids a disable step, and leaves the panel as a screen the compositor keeps committing to. On the owner's machine that is the screen on which commits wait for a page flip that a dark panel never completes. It treats the symptom and keeps its cause.
- **Disable the panel in the shipped configuration.** A fixed `monitor = eDP-1, disable` is right for the owner and wrong for anyone with no external screen, who would have no screen.
- **Disable the panel whenever an external screen is attached, lid or no lid.** People do use both screens with the lid open. The lid is the signal, and the requirement is about the lid.
- **Ask logind for the lid state instead of trusting the event.** It would agree in every case measured, and the binding already says which way the switch moved. A second source would be a second thing to disagree.
- **A listener on the compositor's event socket, to react to a screen being plugged or unplugged.** It is the complete answer to hotplug and it is a daemon, a unit and a dependency on a socket client, for a case the machine's lid behavior probably already covers. See Risks. If the measurement says it does not, that is its own change.

## Risks / Trade-offs

- **Unplugging the external screen with the lid closed.** The panel is off and no screen remains. Nothing here reacts to the unplug, and the expectation is that the system suspends, as it would for any lid-closed laptop that is no longer docked. That expectation is unmeasured and is a task: unplug on the owner's laptop and read what happens. If it does not suspend, the session is left on no output until the lid opens, and the follow-up is the listener above.
- **A reload of the compositor's configuration restores the panel.** The keyword that disables an output is expected not to outlive a reload (not yet measured, and a task reads it), and the compositor reloads when a configuration file changes, which includes the package being upgraded under a running session. The next lid event or the next login corrects it. Accepted, because the failure is toward the panel being on.
- **The initial lid event is a behavior of this compositor version.** It was seen in one log. If a version did not send it, a session that starts with the lid closed would be wrong until the lid moved. The task that exercises the start-up case reads the log for the line rather than relying on this paragraph.
- **A panel named something else.** A built-in panel whose connector does not start with those three prefixes is treated as external and never disabled. That is the safe error.
- **The windows that were on the panel.** The compositor is expected to move a disabled output's workspaces to a remaining one. Not measured; the headless test reads where the windows are.

## Open Questions

- What the system does when the external screen is unplugged with the lid closed. Needs the owner's laptop.
- Whether to re-evaluate when the compositor's outputs change, which the listener alternative would give. Decided by the answer above.
