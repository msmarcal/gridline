# Spec Delta

## MODIFIED Requirements

### Requirement: The session is lockable and the lock survives suspend

A Gridline session SHALL provide screen locking, and a locked session SHALL require authentication before access is restored, including after suspend and resume. The session SHALL lock on its own after a period of inactivity, rather than only when the user asks, and while it is locked the system's own view of the session SHALL agree that it is.

#### Scenario: Manual lock requires authentication

- **WHEN** a user locks a running Gridline session
- **THEN** the session contents are no longer accessible, and authentication is required to return to the desktop

#### Scenario: The session locks itself when idle

- **WHEN** a Gridline session has been inactive for the configured period
- **THEN** the session locks and authentication is required to return to the desktop, without the user having invoked the locker

#### Scenario: The system agrees that the session is locked

- **WHEN** a Gridline session is locked, whether by the user or by inactivity, and the lock component is running
- **THEN** the session manager reports that session as locked, so that a component acting on the system's view of lock state behaves the same way as one acting on the compositor's

#### Scenario: A dead lock component is not reported as a session that is safe to treat as unlocked

- **WHEN** the lock component dies while the compositor holds the session locked
- **THEN** the session manager's view of the session SHALL NOT claim a lock that no component is maintaining, and the session SHALL remain locked, so that the two disagree in the direction that leaves the session protected rather than the direction that lets something skip locking it

#### Scenario: Lock holds across suspend

- **WHEN** a locked Gridline session suspends and later resumes
- **THEN** the session is still locked and authentication is required

#### Scenario: Lock failure does not expose the session

- **WHEN** the lock component fails to start or crashes while the session is locked
- **THEN** the session is not left unlocked and unattended, and the failure is recorded in the system journal

#### Scenario: A dead locker is recoverable without another machine

- **WHEN** the lock component dies while the session is locked and the session is correctly held locked
- **THEN** the user can restore a working lock screen and return to their session from the machine itself, rather than the session being unrecoverable without ending it
