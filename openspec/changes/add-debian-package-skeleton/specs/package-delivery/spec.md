# Spec Delta

## ADDED Requirements

### Requirement: A Gridline package declares only Gridline-named paths

A Gridline package SHALL declare ownership only of paths whose name identifies them as Gridline's, and SHALL NOT declare a path an archive package owns.

Declared ownership is what a package's own install manifests claim. It is not the same as the directory list `dpkg` records, which necessarily includes the shared parents of any nested path.

#### Scenario: Every declared path is Gridline-named

- **WHEN** the install manifests of the Gridline packages are read
- **THEN** every path they declare carries a Gridline-identifying name, and none of them is a directory an archive package owns

#### Scenario: A declared path has exactly one Gridline owner

- **WHEN** the owning package of a Gridline-named directory is queried on an installed system
- **THEN** exactly one Gridline package is named, and it is the package the capability says owns that artifact

#### Scenario: Declaring a path another package owns is caught before release

- **WHEN** a Gridline install manifest would declare a path an archive package owns
- **THEN** the condition is reported during build or review, since installing it produces neither a conflict nor an error and would otherwise go unnoticed

#### Scenario: Influence on a third-party component uses a drop-in

- **WHEN** Gridline needs a third-party component to behave differently
- **THEN** it ships a Gridline-named file into a directory that component reads, and modifies or replaces no file another package owns

### Requirement: Removing a Gridline package leaves shared directories intact

Removing a Gridline package SHALL remove the Gridline-named paths it declared and SHALL leave every shared directory in place, still owned by the archive packages that own it.

This is what protects the fallback: the stock session reads several of those directories, so a removal that took one with it would break the desktop a user removes Gridline to get back to.

#### Scenario: Shared directories survive removal

- **WHEN** every Gridline package is removed from a system where the stock session is installed
- **THEN** the shared directories the packages installed into still exist, and querying their owners returns the same archive packages as before Gridline was installed

#### Scenario: Gridline-named paths do not survive removal

- **WHEN** every Gridline package is removed
- **THEN** no Gridline-named directory the packages declared remains on the system

#### Scenario: The stock session still starts after removal

- **WHEN** a user removes Gridline because the session misbehaved
- **THEN** the stock session starts normally, since no shared directory it depends on was altered or removed
