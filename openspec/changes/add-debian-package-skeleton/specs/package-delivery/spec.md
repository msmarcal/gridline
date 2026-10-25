# Spec Delta

## ADDED Requirements

### Requirement: A Gridline package claims only Gridline-named paths

A Gridline package SHALL declare ownership only of paths whose name identifies them as Gridline's. It SHALL NOT claim a directory that an archive package already owns, and SHALL influence a third-party component by placing a Gridline-named file into a directory that supports one.

#### Scenario: A shared system directory is not claimed

- **WHEN** the owning packages of a directory that archive packages already own are queried
- **THEN** no Gridline package is among them

#### Scenario: A Gridline-named directory is claimed and attributable

- **WHEN** the owning package of a Gridline-named directory is queried
- **THEN** exactly one Gridline package is named, and it is the package the capability says owns that artifact

#### Scenario: Influence on a third-party component uses a drop-in

- **WHEN** Gridline needs a third-party component to behave differently
- **THEN** it ships a Gridline-named file into a directory that component reads, and modifies or replaces no file another package owns

#### Scenario: Claiming a shared directory is caught before release

- **WHEN** a Gridline package would declare a directory an archive package already owns
- **THEN** the condition is reported during build or review, since installing it produces neither a conflict nor an error and would otherwise go unnoticed
