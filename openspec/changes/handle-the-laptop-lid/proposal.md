# Proposal

## Why

On a laptop with its lid closed and an external monitor attached, a Gridline session looks dead. The compositor keeps the closed laptop panel as a screen and makes it the first one, so the cursor, the keyboard focus and every new window go to a panel nobody can see, and the external monitor shows a wallpaper and nothing else. The first run on real hardware failed this way: the owner always works lid closed, and "I could not do anything" was this.

GNOME takes the laptop panel out when the lid closes with another screen attached. Gridline does not, and `platform-integration` already requires lid handling that leaves the session with a usable output.

## What Changes

**This change ships implementation.**

- Takes the laptop's own panel out of the session when the lid is closed and another screen is attached, at session start and when the lid moves during the session, and brings it back when the lid opens.
- Never takes it out when no other screen is attached, so a session is never left without an output by this change. In that case the machine's own lid behavior (suspend, by default) decides.
- Splits the work along the project's boundary: the decision is a compositor-neutral program, and the one action that is compositor-specific, enabling or disabling a named output, is a new verb of the adapter.
- Documents what a person with a docked laptop can expect, and records what was not measured.

**Out of scope.**

- Choosing which screen is primary, positioning or scaling outputs, or any other display layout. Only the lid is handled.
- Reacting to an external screen being unplugged while the lid is closed. See the design: the machine's lid behavior is expected to suspend, which is to be measured on the hardware, and it is not claimed here.
- Per-device hardware automation. The lid is a mandatory integration of the platform, not one device's quirk, which is why it is in scope.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

- `platform-integration`: one requirement **added**. The existing power-management requirement already says an attached external display must not leave the session without usable output on lid close, and says nothing about the case that actually failed: that the panel the lid has covered must stop being treated as a screen.

## Impact

- `packages/gridline/src/adapters/hyprland/adapter`: a new verb that enables or disables a named output.
- `packages/gridline/src/adapters/hyprland/defaults.conf`: two bindings on the lid switch, active while the screen is locked.
- `packages/gridline/src/session/`: a new program that decides, installed with the session package.
- `docs/compositor-adapter.md`, `docs/user-experience.md`, `docs/recovery.md`: the verb, the behavior, and what to do if a screen stays dark.
- Verification: the VM has no lid and one output, so the compositor's real output enable and disable is driven on a headless output, the policy is driven against a fake connector tree, and the whole path is left to the owner's laptop, which is where the defect was found.

## Prior measurement

Measured from three logins on the owner's laptop (Intel UHD 620 and NVIDIA MX150, lid closed, an LG 2560x1080 monitor on `HDMI-A-2`), before writing anything.

**The compositor was healthy and the input arrived.** In the compositor's own log, the session started, found `eDP-1` and `HDMI-A-2`, and modeset both. Touchpad events reached it. The terminal key bound to `SUPER Enter` started the terminal six times in the first login. The owner saw the wallpaper and nothing else.

**With the laptop panel off, it works.** Started from a text console with only `HDMI-A-2` enabled and the lid closed, the owner saw the bar and the cursor and the terminal key opened a terminal. With the panel enabled, in both the real login and a laptop-panel-only run, the log shows `Cannot commit when a page-flip is awaiting` on `eDP-1`; with it disabled the error does not appear.

**The compositor reports the lid itself.** Its log names a `Lid Switch` device and, at startup, `Switch Lid Switch turn on, triggering binds`, with the lid closed. So a binding on the lid switch runs at start as well as on every change, and nothing has to ask the lid's state separately at startup.

**The kernel agrees about the lid, and GNOME had already done this.** `/proc/acpi/button/lid/*/state` reads `closed`, and `xrandr` under GNOME lists only the external monitor, although `eDP-1` is connected, because GNOME had disabled the panel.
