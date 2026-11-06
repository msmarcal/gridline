# Spec Delta

## ADDED Requirements

### Requirement: The login screen offers Gridline and not the compositor it is built on

Where a Gridline installation brings a compositor whose package registers its own session entry, the login screen SHALL offer Gridline and SHALL NOT also offer that compositor's entry, because the entry is present as a consequence of Gridline's dependency rather than a choice the user made. Removing Gridline SHALL restore it.

#### Scenario: One tiling session is offered

- **WHEN** Gridline is installed and the login screen lists the available sessions
- **THEN** Gridline is offered, the stock Ubuntu session is offered, and the compositor's own entry is not

#### Scenario: The compositor's entry returns when Gridline goes

- **WHEN** Gridline is removed from a machine where the compositor remains installed
- **THEN** the login screen offers the compositor's entry again, and the file it reads is the one the compositor's package shipped

#### Scenario: The fallback session is never hidden

- **WHEN** the mechanism that hides a session entry is applied
- **THEN** the stock Ubuntu session entry is not among those hidden, since every recovery path depends on being able to select it, and a machine whose only remaining entry is Gridline has no route back when Gridline does not start

#### Scenario: The compositor stays usable to someone who wants it

- **WHEN** a user wants the compositor's own session on a machine with Gridline installed
- **THEN** the compositor remains installed and startable, and restoring its entry is documented, since this hides an option rather than removing a capability
