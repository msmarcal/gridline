# Spec Delta

## ADDED Requirements

### Requirement: Every relied-on portal interface resolves to a backend Gridline names

Gridline SHALL name a backend for every portal interface a Gridline session relies on, including an interface no installed backend claims under Gridline's desktop identity, so that no relied-on interface is left to the portal frontend's fallback behaviour.

#### Scenario: No installed backend claims the interface

- **WHEN** a portal interface the session relies on is implemented by an installed backend whose own matching rules do not select it for Gridline's desktop identity
- **THEN** Gridline's configuration names that backend for that interface, so the interface resolves by Gridline's choice rather than by whatever the frontend does when nothing matches

#### Scenario: An interface is served by the backend Gridline named

- **WHEN** an application requests a relied-on portal interface in a Gridline session
- **THEN** the backend that serves it is the one Gridline's configuration names for that interface

#### Scenario: Resolution does not depend on the compositor's own identity

- **WHEN** the portal frontend resolves a backend for a relied-on interface
- **THEN** it matches Gridline's own desktop name, and removing the compositor package's claim on that interface would not change which backend serves it

#### Scenario: A relied-on interface with no named backend

- **WHEN** a portal interface the session relies on has no backend named in Gridline's configuration
- **THEN** the condition is treated as a configuration fault in Gridline rather than as acceptable fallback, whether or not the frontend happens to resolve something
