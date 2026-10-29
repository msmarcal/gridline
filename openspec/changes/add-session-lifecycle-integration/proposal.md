# Proposal

## Why

The session starts, and it starts without being able to lock. The adapter reports `lock-signaling`, which says the compositor implements `ext_session_lock_manager_v1`. It does not say a locker is installed, and none is. A desktop that reports it can signal a lock component it does not have is a desktop that cannot lock, which `docs/architecture.md` classifies as a reason to refuse startup rather than a missing feature.

Two blocking components have been recorded without an owner since the component matrix was written: the screen locker and the polkit authentication agent. `docs/component-ownership.md` names candidates for both and assigns the selection here. This change makes the selection and declares it.

The matrix also records why the declaration alone is not enough. The archive's user units carry `PartOf=graphical-session.target` and `Restart=on-failure`, so a failing one restarts quietly and the session comes up regardless. Systemd will not refuse a session for a missing lock component, and the launcher gates only on what the adapter reports. Between those two facts sits the gap this change closes: a component the project calls blocking, that nothing blocks on.

## What Changes

**This change ships implementation.** It selects three components, declares them, and extends the launcher's gate to cover blocking components that live outside the adapter.

- Selects the screen locker, the idle daemon and the polkit authentication agent, with the comparison that decided each recorded in design.md.
- Declares all three as dependencies of `gridline-session`, since each one's absence blocks startup.
- Extends the launcher to verify blocking components outside the adapter before starting the compositor, in the same phase as the existing file-level checks, so the failure costs nothing and names its cause.
- Records the selections in `docs/component-ownership.md`, replacing the rows that name candidates and the table that lists both as deferred without an owner.
- Verifies on the baseline: manual lock requires authentication, the lock survives suspend, suspend and resume return a working session, lid handling and battery observability behave as on a stock Ubuntu desktop, and a polkit prompt can be answered.

**Out of scope.** No lock policy is configured: when the screen locks and how long idle takes belong to `add-gridline-configuration-defaults` and to the policy layer `session-manageability` specifies. The diagnostic that reports a missing component after the fact belongs to `add-session-health-check`; this change makes the condition refuse startup, which is a different mechanism from reporting it. No systemd unit is written or modified, and no power management is implemented, since `systemd-logind` and `upower` already provide it and Gridline's obligation is to verify rather than reimplement.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

- `desktop-session`: gains one requirement, that the session refuses to start when a component the project records as blocking is absent, whether or not that component is reached through the adapter.

`platform-integration` is not modified. It already requires the session to be lockable with the lock surviving suspend, privileged actions to be authorizable from within the session, and power management to behave as on a stock Ubuntu desktop. Those nine scenarios are what this change implements.

The gap is narrow and the matrix already half states it. `compositor-abstraction` makes the session refuse to start when a required **capability** is unreported, and the capability set is what an adapter answers for. The screen locker and the polkit agent are not adapter capabilities; they are components beside the compositor, and nothing requires the session to check them. The matrix records that systemd will not do it either.

## Impact

- `packages/gridline/src/session/gridline-session`: a check for blocking components outside the adapter.
- `packages/gridline/debian/control`: three dependency fields on `gridline-session`.
- `docs/component-ownership.md`: the locker and agent rows, and the deferred-declarations table.
- `docs/security-and-manageability.md`: the passage analysing the candidate locker's PAM stack, which named a component this change does not select.
- Verification runs on the LXD VM where the session starts, and includes suspend, which the VM supports.
