# Design

## Context

See proposal.md - Why. `openspec/specs/desktop-session/spec.md` already states what the session must do; this change builds the two files that make six of its seven requirements exercisable.

What the baseline provides, checked rather than assumed:

- **GDM reads `/usr/share/wayland-sessions/`**, which currently holds exactly one entry, `ubuntu.desktop`, owned by `ubuntu-session`.
- **`DesktopNames` is how the identity is set.** `ubuntu.desktop` carries `DesktopNames=ubuntu;GNOME` and the running session has `XDG_CURRENT_DESKTOP=ubuntu:GNOME`. The display manager performs the translation, semicolons to colons.
- **`TryExec` gates whether an entry is offered.** `ubuntu.desktop` uses it, and an entry whose `TryExec` cannot be resolved is not shown.
- **Hyprland is in the archive at 0.53.3+ds-4 and is not installed.** A launcher that starts a compositor cannot be verified here, which is one reason this change does not try.

## Goals / Non-Goals

**Goals:**

- A session entry GDM offers, named so it does not claim a directory another package owns.
- A launcher that establishes what the session requires, reports the first missing piece by name, and hands the seat back.
- A desktop identity set by the standard mechanism, with one source of truth.
- Verification of the failure path on the baseline, which is the first real test of the fallback guarantee.

**Non-Goals:**

- Starting a compositor, or defining the adapter contract it would speak.
- Any integration beyond the seat: no portals, audio, networking, locking or polkit.
- The diagnostic command. This change writes the record; `add-session-health-check` reads it.

## Decisions

### The session entry is a Gridline-named file in a directory Gridline does not claim

`gridline-session` installs `/usr/share/wayland-sessions/gridline.desktop`. The directory belongs to `ubuntu-session` and is not declared in any Gridline manifest.

*Why:* this is the first exercise of the drop-in rule that `package-delivery` requires. The previous change established the rule by not declaring shared directories; this one is the first to actually place a file in one.

### Desktop identity comes from `DesktopNames`, and this supersedes the archived decision

The entry carries `DesktopNames=Gridline;Hyprland`, which GDM turns into `XDG_CURRENT_DESKTOP=Gridline:Hyprland`. The launcher does not set or modify that variable.

*This changes a decision recorded in `establish-gridline-vision`.* That design said the compositor element would be "supplied by the adapter rather than hardcoded", which implied the launcher composing the value at runtime. Inspecting the baseline showed `DesktopNames` is the mechanism the display manager already uses, and it is static by nature.

*Why supersede it:* one source of truth beats two. If GDM sets the value from the entry and the launcher rewrites it, the identity differs between the moment the session starts and the moment the launcher finishes, and anything reading it in between sees a value that is about to change. Portal backend selection reads this variable, so a window where it is wrong is a window where backend resolution is wrong.

*What it costs:* a second compositor needs a second entry rather than an adapter that reports its name. That is how the archive already handles this case, with Ubuntu shipping separate Wayland and Xorg session entries, and it has a property the dynamic version lacks: the user chooses the compositor at the login screen, visibly, instead of inheriting whatever the adapter resolved.

*Consequence for `add-hyprland-session-baseline`:* the adapter does not supply the identity. If it needs to know which compositor is active, it reads the second element of `XDG_CURRENT_DESKTOP` rather than deciding it.

### `TryExec` names the launcher, so a broken install offers nothing

The entry sets `TryExec` to the launcher path. GDM resolves it before listing the session.

*Why this is a requirement and not only a decision:* the alternative failure is worse than it looks. Without `TryExec`, an installation whose launcher is missing offers a session that dies instantly, and the user cannot tell an incomplete install from a broken desktop. With it, the session simply is not offered, which is a state they can act on.

*Why it is cheap:* GDM already does this for `ubuntu.desktop`, so the behavior is the display manager's and not something Gridline implements.

### The launcher discovers, reports, and exits

The launcher establishes which required components are present. On finding one missing, it writes a record naming the component to the journal under an identifiable Gridline name and exits non-zero, which returns the seat to GDM.

*Why this is the whole launcher for now:* the specs say a launch either reaches a usable desktop or fails visibly. With no adapter, the second branch is the conformant one, and it is the branch the fallback guarantee depends on. Implementing it first means the recovery path is real before anything depends on it.

*Why the record must be attributable now:* `add-session-health-check` reads it. A record that cannot be attributed to Gridline is a record that change cannot find, so the identifier is part of this change's contract rather than a detail to settle later.

*What it does not do:* no fallback to another compositor, no retry, no partial session. A session that cannot be what it claims to be returns control instead of degrading into something unexplained.

### The launcher lives in `/usr/libexec/gridline/`

Installed into the directory `gridline-session` already declares.

*Why not `/usr/bin`:* it is not a command a user runs. `/usr/bin` is for the command surface, which `gridline-tools` owns, and putting a session helper there would cross a boundary `package-delivery` closes with an exclusivity clause.

## Risks / Trade-offs

- **Verification requires logging out** → The failure path can only be exercised from the login screen, so testing it means leaving the current desktop. The expected outcome is a return to GDM, and the stock session is the way back. This is the first change whose verification cannot be done from a terminal.
- **A static `DesktopNames` fixes the compositor in a shipped file** → Supporting a second compositor becomes a second entry. Accepted for one source of truth, with archive precedent, and the cost is visible at the login screen rather than hidden in a resolution order.
- **Teardown is only half exercisable** → There is no logout without a running desktop. The failure path shows the seat being released; the logout path waits for a compositor, which is `add-ci-vm-smoke-tests`'s to confirm end to end.
- **The journal identifier becomes a contract before its consumer exists** → `add-session-health-check` depends on it, so changing it later breaks that change. Naming it here and recording it is cheaper than discovering the coupling later.
- **`TryExec` hides rather than explains** → A user whose entry is withheld sees no Gridline and no error. The requirement's third scenario covers the mitigation, that the path is readable from the entry, but the first signal is an absence, which is a weak signal.

## Migration Plan

Nothing is installed from a previous version, so there is nothing to migrate. The rollback for the verification steps is the fallback itself: select Ubuntu at the login screen, and remove the packages to take the entry with them, which `package-delivery`'s clean-removal requirement already covers and the previous change already verified.

## Open Questions

- Whether the journal record should use a systemd unit or a syslog identifier. Deferred: the requirement asks for attributability, and either satisfies it. `add-session-health-check` is better placed to judge, since it is the consumer.
- Whether a future second compositor gets its own entry or a shared one with a selector. Deferred: it changes nothing until a second adapter exists.
