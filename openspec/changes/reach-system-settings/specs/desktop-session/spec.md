# Spec Delta

## ADDED Requirements

### Requirement: A person can reach the machine's settings from the launcher

A Gridline session SHALL let a person open the machine's settings, including its network and Bluetooth panels, by choosing them in the application launcher, without a terminal and without configuring anything.

#### Scenario: Choosing a settings entry opens it

- **WHEN** a person chooses Settings, Wi-Fi, Bluetooth or Network in the application launcher during a Gridline session
- **THEN** the corresponding panel appears, rather than the launcher closing and nothing opening

#### Scenario: The session's identity is not changed to make it work

- **WHEN** the settings application is started from the launcher
- **THEN** only that program is given an identity it accepts, and every other program started from the launcher, the launcher itself and the session still name Gridline, because other programs and the desktop's portals choose their behavior by that identity

#### Scenario: A panel that cannot start says so

- **WHEN** the settings application fails to start from the launcher
- **THEN** the person is told that it did not open, rather than being left to work out whether the launcher or the machine is at fault

#### Scenario: Another session is unaffected

- **WHEN** a person is in a session other than Gridline's on a machine where Gridline is installed
- **THEN** the settings application starts exactly as it did before Gridline was installed

#### Scenario: Absence degrades and does not block

- **WHEN** the settings application is not installed
- **THEN** the session still starts and the launcher still works, the absence is not reported as a failure of Gridline, and every means of reaching the network and Bluetooth that existed before remains available
