# Spec Delta

## ADDED Requirements

### Requirement: Binary package names carry the gridline- prefix

Every binary package Gridline produces SHALL be named with the `gridline-` prefix.

#### Scenario: Prefix holds for every binary package

- **WHEN** the set of binary packages produced from the Gridline source tree is listed
- **THEN** every name begins with `gridline-`

#### Scenario: Build staging is ignored without a per-package rule

- **WHEN** a new binary package is added and the source tree is built
- **THEN** its build staging directory is already excluded from version control, with no new ignore rule added for it

### Requirement: Dependencies are declared and satisfiable from the supported archive

Each Gridline package SHALL declare the packages it requires, and those declarations SHALL be satisfiable on the supported Ubuntu baseline using the archive alone.

#### Scenario: Install needs no source outside the archive

- **WHEN** a Gridline package is installed on a stock supported Ubuntu system with only the standard archive configured
- **THEN** its declared dependencies resolve and install without adding a third-party repository

#### Scenario: A component Gridline cannot work without is a dependency, not a suggestion

- **WHEN** a component's absence would block the session from starting
- **THEN** it is declared as a dependency rather than as a recommendation or a suggestion

#### Scenario: Archive component is recorded per dependency

- **WHEN** a dependency is declared
- **THEN** whether it comes from main or from universe is recorded, so that the support consequence of depending on it is visible rather than discovered later

### Requirement: Upgrade and downgrade leave a working session and keep local configuration

Upgrading or downgrading Gridline packages SHALL preserve administrator-modified configuration and SHALL leave a session that starts.

#### Scenario: Upgrade preserves local edits

- **WHEN** an administrator has modified a shipped default and the owning package is upgraded
- **THEN** the modification is preserved, and the session starts normally after the upgrade

#### Scenario: Downgrade does not strand the system

- **WHEN** Gridline packages are downgraded to a previously installed version
- **THEN** the session starts normally, and no file from the newer version is left behind that changes its behavior

#### Scenario: A partially completed upgrade is recoverable

- **WHEN** an upgrade is interrupted and leaves packages unconfigured
- **THEN** completing the configuration brings the system to the same state as an uninterrupted upgrade, with no manual file repair

### Requirement: Generated configuration is distinguishable from shipped editable defaults

Gridline SHALL distinguish files an administrator is expected to edit from files Gridline generates, and SHALL NOT present a generated file as hand-maintained.

#### Scenario: A generated file is not registered as a conffile

- **WHEN** the package metadata for a file produced by Gridline's generator is inspected
- **THEN** it is not registered as a conffile, so no upgrade prompt invites an administrator to keep an edit that regeneration will discard

#### Scenario: A generated file is identifiable as generated

- **WHEN** an administrator opens a file Gridline generated
- **THEN** it is identifiable as generated output, naming the supported input to change instead

#### Scenario: Removal takes generated files with it

- **WHEN** the package that owns a generated file is removed
- **THEN** the generated file is removed too, since nothing an administrator authored is lost by removing output

### Requirement: Policy input, generated configuration and lock declarations have named owning packages

Each of policy input, generated configuration and lock declarations SHALL be owned by exactly one named Gridline package, and that ownership SHALL be fixed before any of them ships.

#### Scenario: Each artifact has one owner

- **WHEN** the owning package of policy input, of generated configuration, or of a lock declaration is queried
- **THEN** exactly one Gridline package is named for each, and no two of them are claimed by different packages in different documents

#### Scenario: Ownership does not move after content ships

- **WHEN** a later change adds policy content, generated output or a lock declaration
- **THEN** it installs under the ownership defined here, rather than relocating the boundary
