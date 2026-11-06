# Spec Delta

## MODIFIED Requirements

### Requirement: A Gridline package declares only Gridline-named paths

A Gridline package SHALL declare ownership only of paths whose name identifies them as Gridline's, and SHALL NOT declare a path an archive package owns.

Declared ownership is what a package's own install manifests claim. It is not the same as the directory list `dpkg` records, which necessarily includes the shared parents of any nested path.

A Gridline package SHALL NOT modify or replace a file another package owns. It MAY displace one through a registered diversion, and only where no drop-in path exists for the behaviour required; a diversion SHALL be registered by the package that needs it, SHALL be removed when that package is removed, and SHALL be recorded with the reason it could not be a drop-in.

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

- **WHEN** Gridline needs a third-party component to behave differently and a directory that component reads will carry the change
- **THEN** it ships a Gridline-named file into that directory, and modifies or replaces no file another package owns

#### Scenario: A diversion is used only where a drop-in cannot reach

- **WHEN** Gridline needs a third-party component to behave differently and no directory that component reads is writable by a package
- **THEN** a diversion MAY be used instead, and the reason a drop-in could not serve is recorded where a reader meets the diversion rather than only in the change that introduced it

#### Scenario: A diversion does not outlive the package that needs it

- **WHEN** the Gridline package holding a diversion is removed
- **THEN** the diversion is removed with it and the displaced file is in place again, so no machine is left with an archive package's file relocated by software that is no longer installed

#### Scenario: A diversion covers every file it has to

- **WHEN** the component whose file is diverted ships more than one file serving the same purpose
- **THEN** the diversion covers all of them, and the build or review reports a file that has appeared and is not covered, since one left behind restores the behaviour the diversion exists to prevent
