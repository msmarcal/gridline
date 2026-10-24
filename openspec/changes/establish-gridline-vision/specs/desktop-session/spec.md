# Spec Delta

## Purpose

Gridline's existence as a selectable Wayland desktop session on Ubuntu: how it registers with the display manager, how it starts and stops, how it coexists with the stock GNOME session that remains the user's supported fallback, and how a user recovers when a Gridline session fails.

## ADDED Requirements

### Requirement: Session is selectable in GDM

Gridline SHALL register a Wayland session entry that GDM presents as a selectable option on the login screen once `gridline-session` is installed.

#### Scenario: Session appears after install

- **WHEN** `gridline-session` is installed on Ubuntu 26.04 LTS x86_64 and GDM is restarted or the system is rebooted
- **THEN** GDM's session list offers a Gridline entry alongside the existing Ubuntu sessions

#### Scenario: Session disappears after removal

- **WHEN** `gridline-session` is removed
- **THEN** GDM no longer offers the Gridline entry, and no Gridline session entry remains registered

### Requirement: Session is opt-in, never the default

Installing Gridline SHALL NOT change which session GDM selects by default, for existing users or for users created after installation.

#### Scenario: Existing user's default is preserved

- **WHEN** a user whose last session was Ubuntu GNOME installs `gridline-session` and returns to the login screen
- **THEN** GDM still preselects Ubuntu GNOME, and Gridline is chosen only by explicit selection

#### Scenario: New user does not receive Gridline by default

- **WHEN** a user account is created on a system with Gridline already installed and that user reaches the login screen for the first time
- **THEN** GDM preselects the system's stock Ubuntu session rather than Gridline

### Requirement: GNOME session remains installed and supported

Installing or removing Gridline SHALL leave the regular Ubuntu GNOME session installed, selectable, and functional. Gridline is additive.

#### Scenario: GNOME survives Gridline install

- **WHEN** Gridline is installed
- **THEN** the Ubuntu GNOME session entry is still present in GDM and starts normally when selected

#### Scenario: GNOME survives Gridline removal

- **WHEN** Gridline is removed after use
- **THEN** the Ubuntu GNOME session starts normally, with no leftover Gridline state that alters its behavior

### Requirement: Session launch reaches a usable desktop or fails visibly

Selecting the Gridline session SHALL either reach a usable tiling desktop or return the user to the login screen with the failure recorded in the system journal.

#### Scenario: Successful launch

- **WHEN** a user selects the Gridline session and authenticates
- **THEN** a Wayland tiling desktop starts and accepts keyboard and pointer input

#### Scenario: Launch failure returns control to the user

- **WHEN** a required component of the session fails to start
- **THEN** the session exits, the user is returned to GDM, and the cause is written to the journal under an identifiable Gridline unit or process name

### Requirement: A failed Gridline session is recoverable without Gridline

A user whose Gridline session cannot start SHALL be able to reach a working desktop and locate the failure record without needing a functioning Gridline session.

#### Scenario: Recovery to a working desktop

- **WHEN** the Gridline session fails to start and the user is returned to GDM
- **THEN** the Ubuntu GNOME session remains selectable from the same login screen and starts normally

#### Scenario: Failure is discoverable from the fallback session

- **WHEN** a user logs into the fallback session after a Gridline launch failure
- **THEN** the documented recovery procedure locates the Gridline failure record in the system journal using only tools present on a stock Ubuntu system

### Requirement: Session teardown releases the seat

Logging out of Gridline SHALL terminate the session's processes and return the seat to GDM.

#### Scenario: Clean logout

- **WHEN** a user logs out of a running Gridline session
- **THEN** session processes for that user terminate, and GDM presents the login screen again on the same seat

### Requirement: Session identifies itself to the desktop stack

The running session SHALL advertise a stable desktop identity so that portals, Snap confinement, and application defaults can resolve Gridline-specific behavior.

#### Scenario: Desktop identity is set

- **WHEN** a Gridline session is running
- **THEN** the session environment exposes a stable, documented desktop identity value that does not change between releases without a spec change
