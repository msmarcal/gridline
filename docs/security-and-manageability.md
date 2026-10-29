# Security posture and manageability

This document records what Gridline's security and manageability cost, and where the project cannot promise what a reader might assume. It is not a statement of purpose. Gridline exists to give an Ubuntu user an Omarchy-like tiling session on the machine they already work on; see [vision.md](vision.md).

Reading it as a pitch would be a mistake in both directions. It is not an argument that Gridline is enterprise software, and it is not an argument that security does not matter here. It is the list of facts that constrain the project, written down so that a later change does not have to rediscover them, and so that nobody claims more than the archive can support.

It is a project record; the requirements it implies belong to named changes.

## The target is CIS Benchmark Ubuntu, Level 1 Workstation

The compliance target is deliberately generic: the CIS Benchmark for Ubuntu Linux, Level 1 Workstation, plus screen-lock policy and disk encryption. Not FIPS, not STIG, not PCI. Those regimes demand hardening work and formal validation that a session project cannot deliver on its own, and naming them would promise more than Gridline can test.

Level 1 is the right floor because it is what a mainstream corporate Ubuntu estate is actually measured against, and because its desktop controls are concrete enough to test.

## The security-update gap, stated plainly

Every Hypr-ecosystem package Gridline depends on is in Ubuntu's **universe** component. Universe carries **no Canonical security-update commitment**. Main does.

This is the single largest gap between what Gridline promises and what the archive provides, and it is not closable by packaging work:

- A Gridline installation inherits no CVE-response SLA for its compositor, locker, idle manager or portal backend.
- `gridline-desktop` cannot live in main while it depends on universe, which is the central obstacle to official flavor candidacy.
- The archive also lags upstream: Hyprland 0.53.3 in the baseline against 0.56.2 upstream.

**Gridline's declared position:** the gap is documented, not hidden. Patching responsibility for universe components rests with the operator, and Gridline's documentation must state this where a deploying administrator will read it, not only here. A Gridline-maintained PPA tracking upstream CVEs is recorded as a **future goal**, not a current commitment; promising a patch stream the project cannot yet staff would be worse than naming the gap.

Any Gridline material that claims suitability for a regulated environment without reproducing this gap is wrong, and reviewers should treat it as a defect.

## Central configuration: the mechanism does not exist yet

An enterprise desktop must let an administrator set a policy the user cannot override. On GNOME this is dconf: system databases under `/etc/dconf/db/`, a profile naming their precedence, and a `locks/` subdirectory that makes specific keys unwritable by the user. CIS controls 1.7.4 and 1.7.5 are written directly against that mechanism - 1.7.4 sets `idle-delay` and `lock-delay`, and 1.7.5 locks both so a user cannot change them.

**The compositor does not use dconf.** Verified against the baseline binaries: Hyprland and the tools built on it link `libhyprlang` and resolve configuration through `Hyprutils::Path::findConfig`, with no dconf or GSettings symbols at all. The search order that function implements, read from `hyprutils` source, is:

1. `$XDG_CONFIG_HOME/hypr/<program>.conf`
2. `$HOME/.config/hypr/<program>.conf`
3. each directory in `$XDG_CONFIG_DIRS`, as `<dir>/hypr/<program>.conf`
4. `/etc/xdg/hypr/<program>.conf`

It returns the **first** file it finds and stops. There is no merge, no layering, and no lock concept.

Two consequences follow, and both are structural rather than cosmetic:

- **A user file wins outright.** A `~/.config/hypr/<program>.conf` does not override individual keys from an administrator file; it replaces the administrator file entirely. A setting placed in `/etc/xdg` disappears the moment a user writes their own config, and nothing reports that it happened.
- **CIS 1.7.5 has no counterpart.** "Ensure screen locks cannot be overridden" cannot be satisfied by configuration alone, because the mechanism the control audits is absent from these tools.

**The lock path was kept out of this.** `add-session-lifecycle-integration` selected `swaylock`, which does not link `libhyprlang`, measured against the archive package. The analysis above therefore describes the compositor's own configuration and not the screen lock, which was the point of choosing that way: building the fail-safe component on the model this section identifies as the obstacle would have been a choice, not an inheritance.

`swaylock`'s own configuration model has not been read with the same care, and nothing is claimed about it here.

**The other half of lock policy has no component yet.** Deciding *when* to lock needs an idle daemon, and none is selected: `add-gridline-configuration-defaults` owns that choice along with the policy it serves. One candidate was measured far enough to be worth recording, because it shows the shape of the problem is different rather than absent: `swayidle` reads only user paths, `$XDG_CONFIG_HOME/swayidle/config` and `$HOME/.swayidle/config`, with no system path at all, and its manual states that events given in the config and as arguments are not mutually exclusive, so the two combine rather than one replacing the other. A user can still add events, and the manageability capability still owns making a setting unoverridable.

This is the same class of problem as the portal configuration file, and it has the same shape: the archive provides the component but not the policy surface, so enforcement has to come from something Gridline owns. It is a **deliverable**, not a default.

**What Gridline therefore owes a manageability capability**, as a foundation concern rather than later work:

| Need | Why configuration alone cannot deliver it |
|---|---|
| Vendor, administrator and user precedence | `findConfig` returns the first match and stops; later layers are never read |
| An administrator-locked setting | No lock mechanism exists in `hyprlang`; a user file supersedes wholesale |
| Detection of policy that is not in effect | A superseding user file is indistinguishable from policy being applied |
| A single place an administrator sets policy | Policy for the session, locker, idle manager and portal lives in unrelated files with unrelated formats |

The last row is why this cannot be deferred to the package that ships defaults: deciding where policy lives is an architecture decision, and changing it later invalidates every default already shipped.

That capability is now specified. `openspec/changes/add-manageability-capability/` adds `session-manageability`, which establishes policy input separate from what components read, Gridline-owned generation, per-setting locks, and reporting of policy that is not in effect. The policy layer it implies is recorded in `docs/architecture.md`.

### What the capability makes satisfiable, and what it does not

Claiming a CIS control as satisfied requires enforcement, not detection. The distinction matters because one of Gridline's requirements only reports a condition, and reporting is not compliance.

| Control | Status under Gridline | Mechanism |
|---|---|---|
| 1.7.4, screen locks when idle | Satisfiable | Screen lock policy is set once and generated into both components |
| 1.7.5, screen locks cannot be overridden | Satisfiable only through the policy layer | Per-setting locks, since the dconf lock mechanism the control audits does not exist in these tools |
| Disk encryption | Platform's responsibility | Gridline neither configures nor requires disabling it |
| Secure Boot, TPM | Platform's responsibility | Out of scope, must not break |
| Enterprise identity, including smartcard | Platform's responsibility | Gridline's obligation is non-interference, verified rather than assumed |
| Audit logging | Platform's responsibility | Session components log to the journal and do not bypass `auditd` |

The second row is the one to read carefully. The control is satisfiable, but only because Gridline supplies a mechanism that the underlying tools lack. An auditor comparing a Gridline machine to a GNOME machine will not find the dconf locks the benchmark's audit script looks for, so a Gridline deployment needs its compliance evidence expressed in Gridline's own terms. That is a documentation obligation this project has not yet met, and it is not satisfied by the specs alone.

## Lock screen and idle policy

`swaylock` ships `/etc/pam.d/swaylock` containing `auth include login`, so authentication works with no packaging effort. The rejected candidate shipped the identical stack, so the observation transfers rather than being discarded:

- `include login` is broader than a locker requires. GDM uses a narrower stack. This is a security-review item, not a defect, and it belongs to whichever change hardens the lock path.
- Screen-lock policy spans two components: an idle daemon decides when to lock, `swaylock` decides what unlocking requires. A policy claim about lock timeout is a claim about both, and the diagnostic must check both. Only one of the two is selected, so a timeout claim cannot be made at all yet, which is a more useful thing to know than a claim resting on a component nobody chose.

## Screen capture cuts both ways

Gridline already requires deterministic portal backend selection so screen sharing works. An enterprise target adds the opposite requirement: an administrator may need to **restrict** capture. The portal mechanism serves both, which is an argument for routing capture through portals rather than letting applications reach the compositor directly - a restriction expressed in portal configuration is auditable, one enforced by a compositor flag is not.

## What stays out of scope

Gridline must not require anything outside the supported Ubuntu stack, and must not interfere with mechanisms a corporate estate depends on:

- Full-disk encryption, Secure Boot and TPM behavior are the platform's, not the session's. Gridline neither requires disabling them nor claims to configure them.
- Enterprise identity - SSSD, Kerberos, Active Directory, smartcard login - is a platform concern. Gridline's obligation is to not break it, and to verify that it does not.
- Audit and logging continue to work: the session does not bypass `auditd`, and its components log to the journal.

These are verification obligations, not features. A tiling session that quietly breaks smartcard login is useless to the audience this project is for, and the failure would surface during a pilot rather than during development unless something checks it.

## Reversibility is a compliance argument

GNOME remaining installed, selectable and supported is already a constraint. In an enterprise framing it is also the strongest risk argument Gridline has: a session that can be removed, leaving a supported desktop behind, is a reversible change. That property is worth stating in those terms when the audience is a platform team rather than an enthusiast.
