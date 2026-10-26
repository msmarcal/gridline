# Spec Delta

## ADDED Requirements

### Requirement: A session entry is offered only when its launcher is present

The Gridline session entry SHALL name the launcher it starts in a way the display manager can resolve before offering the session, so that an installation missing the launcher offers no session rather than one that cannot start.

The fallback is the stock session: when the Gridline entry is withheld, the login screen still offers Ubuntu, so a user is never left without a selectable desktop.

#### Scenario: Entry is offered when the launcher is present

- **WHEN** `gridline-session` is fully installed and the login screen is shown
- **THEN** the Gridline entry appears in the session list

#### Scenario: Entry is withheld when the launcher is absent

- **WHEN** the session entry is present but the launcher it names is not
- **THEN** the login screen does not offer the Gridline session, and the stock Ubuntu session is still offered and selectable

#### Scenario: A withheld entry is diagnosable

- **WHEN** the Gridline entry is expected but not offered
- **THEN** the launcher path the entry names can be read from the entry itself, so the missing file is identifiable without guessing
