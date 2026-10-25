## Why

`establish-gridline-vision` defined Gridline as a session that is additive, removable and integrated with Ubuntu's platform components. It did not define how an administrator sets policy that a user cannot silently override, because at the time the project's framing was a tiling session for individual users.

The framing has since been made explicit: Gridline's differentiator is enterprise deployability, with the CIS Benchmark for Ubuntu Linux, Level 1 Workstation as the named compliance target. That creates an obligation the foundation does not carry, and inspecting the baseline showed the obligation cannot be met by configuration alone.

Two facts, both verified against the packages in the Ubuntu 26.04 LTS archive rather than assumed:

**The Hypr tools do not use dconf.** `hypridle` and `hyprlock` contain no dconf or GSettings symbols. They resolve configuration through `Hyprutils::Path::findConfig`, whose search order is `$XDG_CONFIG_HOME`, then `$HOME/.config`, then each entry of `XDG_CONFIG_DIRS`, then `/etc/xdg` — returning the **first** file found and stopping. There is no merge, no layering across directories, and no lock mechanism.

**The consequence is that a user file defeats administrator policy wholesale.** A `~/.config/hypr/hyprlock.conf` does not override individual keys from a Gridline-shipped file; it replaces that file entirely. A lock timeout set by an administrator disappears the moment a user writes any config of their own, and nothing reports that it happened. CIS control 1.7.5, "ensure screen locks cannot be overridden", has no counterpart under this mechanism, because the mechanism the control audits is absent.

This also contradicts a statement already in the project's documentation. `docs/configuration.md` describes precedence as ordinary XDG resolution in which "a user override always wins over a shipped Gridline default". That is correct for XDG-aware applications reading `XDG_CONFIG_DIRS`, and wrong for the `hypr*` configuration files, where the user file does not override a default but replaces the only file that would have been read. The document does not currently distinguish the two cases.

The project's own rules make this a foundation gap rather than future work. `openspec/config.yaml` requires that a requirement depending on a third party honoring Gridline's identity name the mechanism that makes it hold, and requires that enforced policy name the Gridline-owned mechanism that enforces it. No requirement in the foundation does either for configuration policy. Deciding where policy lives is also an architecture decision: every default shipped before it is settled is a default that may have to be re-expressed.

## What Changes

This change adds one capability, `session-manageability`, to the foundation. It produces specifications and documentation only. No package is built and no generator is implemented here.

The capability establishes that Gridline owns a policy input distinct from the configuration files its components read, that a Gridline-owned generator produces the effective configuration from that input, that individual settings can be marked as administrator-locked, and that policy which is not in effect is reported rather than silently superseded.

It also corrects `docs/configuration.md`, which currently describes a single precedence model that does not apply to the `hypr*` files, and adds the policy layer to `docs/architecture.md`.

- **Specs added:** `session-manageability`
- **Docs corrected:** `docs/configuration.md`, `docs/architecture.md`
- **Sequence position:** before `define-debian-package-architecture`, because the package that ships policy and the package that ships defaults both depend on where policy lives
- **Not in scope:** the policy file format, the specific settings an administrator may lock, the generator implementation, and any CIS control mapping beyond the screen-lock controls that motivated the capability

## Requirement attribution

Every requirement in this change is the responsibility of exactly one downstream change. The sequence position matters: the first two must be settled before any package is built, because they decide what the packages contain.

| Requirement | Owned by | Why there |
|---|---|---|
| Policy input is separate from the files components read | `define-debian-package-architecture` | Where policy input lives and which package owns it is a package boundary decision, and moving it later invalidates shipped defaults |
| Effective configuration is produced by a Gridline-owned generator | `define-debian-package-architecture` | The generator's place in the build and the rule that generated files are not conffiles are both packaging decisions |
| An administrator can lock an individual setting | `add-gridline-configuration-defaults` | The lock declaration is part of the configuration surface this change owns, and it decides which defaults are enforceable |
| Screen lock policy is enforceable as a single policy | `add-gridline-configuration-defaults` | Screen lock is a shipped default with a policy dimension, owned alongside the other defaults |
| Policy that is not in effect is reported | `add-session-health-check` | Detecting a silently superseded policy is a health check, and that change owns the diagnostic surface |
| Policy does not alter configuration resolution for other sessions | `define-supported-ubuntu-integration` | Non-interference with the GNOME session is that change's subject, and this is the policy-layer instance of it |

No requirement in this change is deferred. A deferred requirement would have to be listed here explicitly; leaving one out of both this table and a deferral table is an attribution failure, not a deferral.

## Impact

`add-gridline-configuration-defaults` is the change most affected. It was already identified as the change that decides whether the project is good, since every binary comes from the archive unmodified and the configuration is the product. It now has a precondition: the defaults it ships must be expressible as policy input, and any default an administrator is expected to enforce must be lockable.

`add-session-health-check` gains an obligation: detecting that administrator policy is not in effect is a health check, and it is the only mechanism that converts a silent override into a visible fault.

`define-debian-package-architecture` must place policy input, generated configuration and lock declarations in packages whose boundaries do not change later. Generated configuration is not a conffile, which is a distinction that change has to make explicitly.

No requirement in `establish-gridline-vision` is withdrawn or modified. The 32 requirements there remain as specified; this change adds to them.
