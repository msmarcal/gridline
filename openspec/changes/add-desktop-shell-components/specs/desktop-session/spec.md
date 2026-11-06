# Spec Delta

## ADDED Requirements

### Requirement: The session presents a desktop rather than a bare compositor

A Gridline session SHALL show the state of the machine, deliver the notifications applications send, and present a background, without the user installing or configuring anything. None of these components is blocking: each one's absence SHALL degrade the desktop rather than prevent the session from starting, because a desktop that shows less is worse and not unsafe.

#### Scenario: The state of the machine is visible

- **WHEN** a Gridline session is running
- **THEN** the user can see, without running a command, which workspace they are on, the time, and the state of the machine's power source and network

#### Scenario: A notification reaches the user

- **WHEN** an application sends a desktop notification during a Gridline session
- **THEN** the notification is shown to the user and then dismissed or expires, rather than being discarded because nothing is listening for it

#### Scenario: The desktop has a background

- **WHEN** a Gridline session reaches the desktop with no application window covering the screen
- **THEN** a background is displayed, rather than whatever the compositor leaves behind when nothing has been drawn

#### Scenario: Each component is kept running

- **WHEN** one of these components exits during a session, for any reason
- **THEN** it is started again without the user intervening, and the session continues through the gap rather than ending

#### Scenario: An absent component degrades the desktop and does not block the session

- **WHEN** one of these components is not installed, or fails to start
- **THEN** the session still starts and reaches a usable desktop, with the remaining components unaffected, and the condition reported as a capability gap rather than refusing the session

#### Scenario: Their configuration is a shipped default a user can decline

- **WHEN** a user replaces the configuration of one of these components with their own
- **THEN** their configuration applies, exactly as it would on a machine where Gridline had shipped none, and the session does not reimpose Gridline's

#### Scenario: Only the Gridline session is affected

- **WHEN** a user logs into a session other than Gridline on a machine where Gridline is installed
- **THEN** none of these components is started for that session, and that session presents what it presented before Gridline was installed
