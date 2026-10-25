# Spec Delta

## ADDED Requirements

### Requirement: Binary package names carry the gridline- prefix

Every binary package Gridline produces SHALL be named with the `gridline-` prefix.

#### Scenario: Prefix holds for every binary package

- **WHEN** the set of binary packages produced from the Gridline source tree is listed
- **THEN** every name begins with `gridline-`

### Requirement: Dependencies are declared and satisfiable from the supported archive

Each Gridline package SHALL declare the packages it requires, and those declarations SHALL be satisfiable on the supported Ubuntu baseline using the archive alone.

#### Scenario: Install needs no source outside the archive

- **WHEN** a Gridline package is installed on a stock supported Ubuntu system with only the standard archive configured
- **THEN** its declared dependencies resolve and install without adding a third-party repository

#### Scenario: A component Gridline cannot work without is a dependency, not a suggestion

- **WHEN** a component's absence would block the session from starting
- **THEN** it is declared as a dependency rather than as a recommendation or a suggestion

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

### Requirement: Package ownership of a configuration artifact does not move once content ships

Once a Gridline package owns policy input, generated configuration or lock declarations, a later change SHALL add content under that ownership rather than relocating the boundary.

#### Scenario: Ownership does not move after content ships

- **WHEN** a later change adds policy content, generated output or a lock declaration
- **THEN** it installs under the ownership already specified, rather than relocating the boundary

#### Scenario: A removed generator leaves no unowned destination

- **WHEN** the package owning the generator is removed while the package owning the destination stays installed
- **THEN** the destination remains owned and no generated file is left that nothing can regenerate or remove

## MODIFIED Requirements

### Requirement: gridline-config owns defaults

`gridline-config` SHALL own the shipped defaults under `/etc/xdg`, the default keybindings, the default themes, the administrator's policy input, the lock declarations, and the destination of the generated configuration, and nothing else.

#### Scenario: Defaults are owned by gridline-config

- **WHEN** the owning package of a shipped default under `/etc/xdg` is queried
- **THEN** the answer is `gridline-config`

#### Scenario: Local changes survive upgrade

- **WHEN** an administrator has modified a shipped default and `gridline-config` is upgraded
- **THEN** the local modification is preserved rather than silently overwritten

#### Scenario: Policy input and lock declarations are owned by gridline-config

- **WHEN** the owning package of the administrator's policy input or of a lock declaration is queried
- **THEN** the answer is `gridline-config`

#### Scenario: The generated configuration destination is owned, not its content

- **WHEN** `gridline-config` is installed before any configuration has been generated
- **THEN** it owns the destination of the generated configuration and ships no generated content there

### Requirement: gridline-tools owns the command-line surface

`gridline-tools` SHALL own `gridline doctor`, the theme and diagnostic commands, and the generator that produces the effective configuration, and nothing else.

#### Scenario: Commands are owned by gridline-tools

- **WHEN** the owning package of the `gridline` command is queried
- **THEN** the answer is `gridline-tools`

#### Scenario: Tools are independently installable

- **WHEN** `gridline-tools` is installed without a Gridline session running
- **THEN** its commands run and report on the system rather than failing to start

#### Scenario: The configuration generator is owned by gridline-tools

- **WHEN** the owning package of the generator that produces the effective configuration is queried
- **THEN** the answer is `gridline-tools`

### Requirement: gridline-desktop is reserved as a metapackage boundary

The name `gridline-desktop` SHALL be reserved for a future metapackage that pulls in the packages making up a complete Gridline desktop, and SHALL NOT be delivered by the change that reserves the name. When delivered, it SHALL contain no functionality of its own.

#### Scenario: Reserved name is not delivered prematurely

- **WHEN** the set of binary packages Gridline currently delivers is inspected
- **THEN** no `gridline-desktop` package is present

#### Scenario: Metapackage ships no implementation when delivered

- **WHEN** a future change delivers `gridline-desktop` and it is inspected
- **THEN** it ships no executable, configuration, or data files beyond its own packaging metadata
