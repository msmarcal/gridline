# Design

## Context

See proposal.md for motivation. Four measurements and one rule constrain the approach.

**The entry is Gridline's side effect, not a user's choice.** `apt-mark showmanual` on the baseline does not list the compositor; it arrived as a dependency of `gridline-session`. This is the whole argument for acting at all, and it is a measurement rather than a view.

**There are two entries, not one.** The compositor's package ships `hyprland.desktop` and `hyprland-uwsm.desktop`. The second is absent from the login screen today only because its `TryExec` names `uwsm`, which the baseline does not install. That is a fact about the baseline, so a mechanism covering the first and not the second works until the day something installs `uwsm`.

**The display manager resolves session entries by filename, first match wins.** Measured by placing a file of the same name in a higher-precedence directory: the entry shown was the one from that directory, not both. The display manager also honours `Hidden=true` in a session entry, measured the same way, which is what makes a masking file able to remove an entry rather than replace it.

**The masking directory is not available to a package.** The only directory with precedence over `/usr/share/wayland-sessions` is `/usr/local/share/wayland-sessions`, which Debian Policy §9.1.2 reserves for the local administrator and forbids any package from writing to. The clean mechanism exists and belongs to someone else.

**An active requirement forbade this.** `package-delivery` required that influencing a third-party component use a drop-in and modify or replace no file another package owns. That requirement is right about the general case and was written before a case appeared that no drop-in can serve. The delta modifies it to admit one bounded exception rather than weakening it.

## Goals / Non-Goals

**Goals:**

- One tiling session on the login screen, the one the user installed.
- A machine that loses Gridline gets the compositor's entry back, with the compositor's own file in place.
- An exception to a rule that is narrower than the rule, and carries obligations the rule did not need.

**Non-Goals:**

- Hiding anything else, and the stock session in particular.
- Making the compositor unreachable. This hides an option; the compositor stays installed and startable.
- Closing the configuration leak. Diverting the entry removes the accidental path into a half-Gridline session and leaves untouched the reason that session was half-Gridline, which is recorded and owned elsewhere.

## Decisions

### `dpkg-divert`, because the alternative is not available to a package

A diversion is what Debian provides for one package displacing another's file, and it has the property this needs: the compositor's package keeps owning its file, upgrades of that package continue to work and land at the diverted path, and removing the diversion puts the file back without anyone having to have kept a copy.

The alternative that costs nothing is a masking file in a higher-precedence directory, and it works: measured, a file of the same name in `/usr/local/share/wayland-sessions` replaces rather than joins the packaged one, and `Hidden=true` in it removes the entry from the selector. That is the recipe `docs/recovery.md` documents for an administrator, and it is the right mechanism in the wrong hands: Policy forbids a package from writing there. So the package cannot use the clean route, and the route it can use is the heavier one.

Editing or deleting the compositor's file was never a candidate. It breaks on upgrade, it makes `dpkg --verify` fail for a package Gridline does not own, and it leaves nothing to restore.

### Maintainer scripts, which this project has not needed until now

`gridline-session` gains a `postinst` and a `postrm`. They are the first maintainer scripts in the project, and that is worth noting because `package-delivery` has a requirement that nothing is fetched at install time and a general preference for packages that are inert. A diversion cannot be expressed in an install manifest, so this is the one thing that has to happen at install time rather than at build time.

The risk is entirely in the `postrm`. A `postinst` that fails leaves an install error, which is loud. A `postrm` that fails to remove the diversion leaves an archive package's file relocated on a machine Gridline is no longer on, which is silent and is a worse failure than anything this change is trying to fix. The delta therefore carries it as a scenario rather than leaving it to care, and the verification exercises removal rather than reasoning about it.

### Covering every entry, and noticing a new one

Diverting one of two entries produces a mechanism that works until `uwsm` is installed and then quietly does not. Both are diverted.

That still leaves the case where the compositor's package gains a third entry in a later version. A diversion names a path, so a new path is simply not covered, and nothing reports it. The delta asks that the build or review report an uncovered entry, which means something has to enumerate what the compositor's package ships and compare it with what the diversion covers. That check is the part of this change most likely to be skipped and the part that keeps it working a year from now.

### The fallback is never a candidate

The mechanism is general: it hides whatever entry it is pointed at. Pointed at the stock session it would produce a machine whose only remaining option is Gridline, which contradicts every recovery path the project documents and would make a Gridline that fails to start into a machine with no desktop.

This is a scenario rather than a comment because the mechanism arrives in the same change as the judgement about where to apply it, and the judgement is the part that does not survive being obvious.

## Risks / Trade-offs

- **A second obstacle to official flavor candidacy.** → Accepted deliberately, recorded in `docs/security-and-manageability.md` beside the universe obstacle it joins. A package diverting another maintainer's file by default is a serious review objection independent of the first.
- **A user who later wants the compositor's own session finds its entry gone.** → The compositor is installed and startable, and `docs/recovery.md` documents restoring the entry. The cost is that they must discover this; the benefit is that everyone else is not offered a session they did not install.
- **The project gains maintainer scripts, and with them a class of failure it has not had.** → One script pair, doing one thing, with the removal path carried as a scenario and exercised rather than argued.
- **The modified requirement could be read as permission to divert more.** → The modification is written as a bounded exception with three obligations attached, and the scenario requiring a drop-in where one can reach is kept unchanged beside it.

## Open Questions

- Whether `dpkg-divert` with `--rename` and no replacement file is the right shape here, given that Gridline supplies nothing at the diverted path and only wants the original gone. Measurable before implementation.
- What enumerates the compositor's session entries for the coverage check: the package's file list at build time names what the baseline ships, and a check against the installed package names what the machine has. These answer different questions and the change should say which one it is making.
