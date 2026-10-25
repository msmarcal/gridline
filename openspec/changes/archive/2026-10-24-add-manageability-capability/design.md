# Design

## Context

Gridline's components do not share a configuration mechanism with the GNOME desktop, and the difference is not cosmetic. GNOME exposes policy through dconf, where a system database plus a `locks/` directory lets an administrator set a value and make it unwritable. The CIS Benchmark for Ubuntu writes its desktop controls directly against that mechanism: control 1.7.4 sets `idle-delay` and `lock-delay`, and control 1.7.5 locks both.

The baseline components provide no equivalent. Verified against the packages in the Ubuntu 26.04 LTS archive:

- `hypridle` and `hyprlock` contain no dconf or GSettings symbols.
- Both resolve configuration through `Hyprutils::Path::findConfig`.
- That function checks `$XDG_CONFIG_HOME`, then `$HOME/.config`, then each entry of `XDG_CONFIG_DIRS`, then `/etc/xdg`, and returns the first match.

The consequence is the design problem: a user file does not override individual settings from an administrator file, it replaces the file entirely. There is no layering to extend and no lock to set.

## Goals / Non-Goals

**Goals.** Establish that Gridline owns a policy input distinct from what components read; that generation is Gridline's responsibility rather than a third-party loader's; that individual settings can be locked; and that policy which is not in effect is reported rather than silently superseded.

**Non-goals.** The policy file format, the specific settings that may be locked, the generator's implementation, the CIS control mapping beyond screen lock, and anything about GNOME's own policy mechanism. Gridline does not change how dconf works and does not route its own policy through it.

## Decisions

### Policy is an input, generated configuration is an output

The alternative was to keep treating shipped defaults as the policy surface and document that users can defeat them. That is what the project did implicitly until the enterprise framing was made explicit, and it fails the stated target: an administrator cannot enforce a screen lock timeout, and cannot tell that theirs is not in effect.

Separating input from output costs a generator and a build-time step that did not exist before. It buys the only thing that makes enforcement possible: a point where Gridline decides what the component will read, rather than hoping precedence decides it correctly.

**Consequence for packaging:** generated configuration is not a dpkg conffile. Conffiles exist so local edits survive upgrade, which is right for a default an administrator is expected to edit and wrong for a file that may be regenerated. `define-debian-package-architecture` has to make that distinction explicitly, because the natural instinct for a file under `/etc` is to register it.

### Locking is declared, not implied by location

A lock could have been implied — for example, treating anything in a particular directory as locked. That was rejected because it makes the lock set invisible at the point an administrator reads policy, and because it couples enforcement to installation layout, which packaging changes later.

An explicit declaration means the answer to "what can a user not change on this machine" is a file an administrator can read, rather than an inference from directory structure.

### Detection is a requirement, not a diagnostic nicety

The requirement that unapplied policy be reported is the one that turns the rest from documentation into something testable. Without it, every other requirement here can be satisfied on a machine where policy silently does nothing, and nobody would know.

This is the same lesson the portal configuration file produced: a component that is installed but not selected looks identical to one that is working, and the fix is a check that distinguishes them. The difference is that the portal case has a worse variant — a missing Gridline file is masked by the compositor's own file, so capture works for the wrong reason. Policy has no such accidental fallback, which makes it easier to detect and more important to check.

### Screen lock is one policy across two components

`hypridle` decides when to lock and `hyprlock` decides what unlocking requires. Treating them as two policies would let a machine be configured with a locked timeout and an unlocked locker, which satisfies neither the control nor common sense. Expressing it once is also what makes the CIS control legible: the control is about the behavior, not about which process implements it.

## Risks / Trade-offs

**A generator is code Gridline must maintain**, in a project whose stated advantage is shipping almost no code. This is accepted because the alternative is not less code, it is an unenforceable claim. It should stay as small as the requirements allow, and the specs deliberately avoid mandating a format or an implementation that would grow it.

**Policy adds a failure mode that did not exist**: generation can fail, or produce a file a component rejects. The reporting requirement covers it, and `add-session-health-check` owns surfacing it, but the risk is real and worth naming rather than discovering later.

**Upstream could add a lock mechanism**, which would make part of this layer redundant. That would be a good outcome. The specs are written against the behavior required, not against the absence of an upstream feature, so an upstream mechanism becomes an implementation option rather than a reason to respecify.

## Open Questions

- Whether policy input is one file or one file per component. One file is better for the administrator and worse for the generator; the specs do not decide it.
- Whether a locked setting that the user has also set should be reported as a conflict or silently take precedence. The specs require the locked value to win and require unenforceable locks to be reported, but do not say whether an overridden user value deserves a message.
- Whether policy should be expressible per user or per group, which every real enterprise eventually asks for and which this change deliberately leaves out of scope.
