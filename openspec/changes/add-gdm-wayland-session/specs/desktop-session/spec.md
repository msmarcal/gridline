# Spec Delta

## MODIFIED Requirements

### Requirement: Session launch reaches a usable desktop or fails visibly

Selecting the Gridline session SHALL either reach a usable tiling desktop or return the user to the login screen with the failure recorded in the system journal.

The record SHALL be selectable by its author rather than by searching the journal's text, because a user reading it has just been returned to the login screen and knows neither which boot the failure belongs to nor how recently it happened.

#### Scenario: Successful launch

- **WHEN** a user selects the Gridline session and authenticates
- **THEN** a Wayland tiling desktop starts and accepts keyboard and pointer input

#### Scenario: Launch failure returns control to the user

- **WHEN** a required component of the session fails to start
- **THEN** the session exits, the user is returned to GDM, and the cause is written to the journal under the syslog identifier `gridline-session`

#### Scenario: The failure record is selectable without a time window

- **WHEN** a user returned to the login screen looks for the cause from the stock session
- **THEN** selecting records by the syslog identifier returns the launcher's records and no others, without naming a boot or a time range

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
