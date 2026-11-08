# Spec Delta

## ADDED Requirements

### Requirement: An application can be started without a terminal

A Gridline session SHALL let a user find an installed application by name and start it, from the keyboard, without opening a terminal and without configuring anything. The component is not blocking and runs on demand.

#### Scenario: An installed application can be found and started

- **WHEN** a user asks for the application launcher during a Gridline session and types part of an installed application's name
- **THEN** that application is offered and selecting it starts it, without the user naming a path or a package

#### Scenario: Reaching it needs no pointer and no prior knowledge of a command

- **WHEN** a user has a Gridline session and a keyboard
- **THEN** the application launcher is reachable by a key combination the session ships, and that combination is documented in a file the user can read without the session running

#### Scenario: Not running is not a fault

- **WHEN** no application launcher is on screen during a healthy Gridline session
- **THEN** the session reports the component as installed and able to run rather than as stopped or failed, so that the usual state of an on-demand component does not read as a defect

#### Scenario: An absent launcher degrades the desktop and does not block the session

- **WHEN** the application launcher is not installed, or fails to start
- **THEN** the session still starts and reaches a usable desktop, and the condition is reported as a capability gap rather than refusing the session

#### Scenario: The session's binding does not name the launcher it ships

- **WHEN** the shipped compositor configuration is read to find what the application launcher binding runs
- **THEN** it names a Gridline-owned program rather than the launcher's own executable, so that replacing the launcher is a change to one Gridline file and not to a user's keybindings

#### Scenario: Its configuration is a shipped default a user can decline

- **WHEN** a user replaces the application launcher's configuration with their own
- **THEN** their configuration applies, exactly as it would on a machine where Gridline had shipped none, and the session does not reimpose Gridline's

#### Scenario: Only the Gridline session is affected

- **WHEN** a user logs into a session other than Gridline on a machine where Gridline is installed
- **THEN** nothing Gridline ships for the application launcher changes what that session does, and that session presents what it presented before Gridline was installed

### Requirement: The session's own actions are reachable without knowing their commands

A Gridline session SHALL present its own actions - locking the screen, ending the session, suspending, restarting and powering off - so that a user can perform each one without knowing a key combination, a command, or which program implements it. The component is not blocking and runs on demand.

#### Scenario: Each action is listed and can be chosen

- **WHEN** a user asks for the command menu during a Gridline session
- **THEN** locking, ending the session, suspending, restarting and powering off are each listed in words that say what they do, and choosing one performs it

#### Scenario: An action asked for here is the same action asked for elsewhere

- **WHEN** the command menu's lock entry is chosen
- **THEN** the session is locked by the same path a lock requested from the keyboard, the idle timeout or another program takes, rather than by the menu starting a locker of its own, because `session-manageability` requires lock policy to be one policy and not several components configured independently

#### Scenario: Ending the session releases the seat

- **WHEN** the command menu's entry for ending the session is chosen
- **THEN** the session ends and the seat is returned to the display manager, which is the behavior `desktop-session` already requires of session teardown however it was triggered

#### Scenario: An action that cannot be performed says so

- **WHEN** an action in the menu is chosen and the system refuses it, or the component that performs it is unavailable
- **THEN** the user is told that it did not happen, rather than the menu closing and leaving them to infer the outcome from the machine's behavior

#### Scenario: Losing it costs no action

- **WHEN** the command menu is not installed or fails to start
- **THEN** every action it offers remains available by the means that existed before it - a key combination or a command - and the session still starts, with the condition reported as a capability gap

#### Scenario: Only the Gridline session is affected

- **WHEN** a user logs into a session other than Gridline on a machine where Gridline is installed
- **THEN** nothing Gridline ships for the command menu changes what that session does
