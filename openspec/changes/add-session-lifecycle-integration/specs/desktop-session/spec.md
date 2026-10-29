# Spec Delta

## ADDED Requirements

### Requirement: The session refuses to start when a blocking component is absent

A session SHALL NOT start when a component the project records as blocking is absent, whether or not that component is reached through the compositor adapter, and the absent component SHALL be named in the failure record.

#### Scenario: A blocking component outside the adapter is absent

- **WHEN** a component the project records as blocking startup is not installed, and that component is not one the adapter reports on
- **THEN** the session does not start, the component is named in the journal under the session's syslog identifier, and the user is returned to the login screen with the stock Ubuntu session still selectable

#### Scenario: The check runs before the compositor

- **WHEN** a blocking component is absent
- **THEN** the launch ends before a compositor process is created, since nothing has to run for an absent file to be observed

#### Scenario: A capability is not a component

- **WHEN** the adapter reports the capability through which a blocking component is used, and that component is not installed
- **THEN** the session still refuses to start, because a compositor that implements a protocol is not the same thing as the component that speaks it

#### Scenario: Restarting units do not satisfy the rule

- **WHEN** a blocking component ships a service unit that restarts on failure
- **THEN** the session's own check decides whether to start, rather than treating a unit that will retry as evidence the component is present
