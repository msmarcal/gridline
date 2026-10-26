# Spec Delta

## ADDED Requirements

### Requirement: A session with a complete adapter reaches a usable desktop

When the active adapter reports every required capability, selecting Gridline SHALL reach a running tiling desktop that accepts input, rather than only failing visibly.

#### Scenario: The session starts

- **WHEN** a user selects Gridline and authenticates on a system whose adapter reports every required capability
- **THEN** a Wayland tiling desktop starts, accepts keyboard and pointer input, and the session is recorded as running for that user

#### Scenario: The compositor fails after the capability check passes

- **WHEN** every required capability is reported but the compositor fails to start or exits unexpectedly
- **THEN** the session ends rather than leaving a seat with no desktop, the cause is written to the journal under the session's syslog identifier, and the user is returned to the login screen with the stock Ubuntu session still selectable

#### Scenario: Logout releases the seat

- **WHEN** a user logs out of a running Gridline session
- **THEN** the compositor terminates, the seat returns to the display manager, and the login screen is usable without a reboot
