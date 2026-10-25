# package-delivery Specification

## Purpose
The contract for how Gridline reaches users: exclusively as Debian packages, with fixed boundaries for what each package owns, so that any delivery vehicle such as a future ISO consumes the same packages rather than carrying functionality of its own.

## Requirements

### Requirement: Functionality ships only in Debian packages

Every user-visible Gridline capability SHALL be delivered by a Debian package. No capability may require an install script, a post-install download, or a manual file copy to work.

#### Scenario: Installed from packages alone

- **WHEN** a user installs the Gridline packages on a stock Ubuntu 26.04 LTS x86_64 system with no additional setup steps
- **THEN** the Gridline session is selectable in GDM and functional

#### Scenario: No out-of-band fetching at install time

- **WHEN** any Gridline package is installed or configured
- **THEN** it does not download code or configuration from the network to complete its own installation

### Requirement: Package boundaries are fixed and non-overlapping

Gridline SHALL be delivered as `gridline-session`, `gridline-config`, and `gridline-tools`, with each file owned by exactly one package. The name `gridline-desktop` SHALL be reserved for a metapackage delivered by a later change.

#### Scenario: No file is claimed twice

- **WHEN** the Gridline packages are built and installed together
- **THEN** no two packages ship the same path, and installation completes without file conflicts

### Requirement: gridline-session owns session integration

`gridline-session` SHALL own the GDM session entry, the session launcher, and the core integration glue, and nothing else.

#### Scenario: Session entry is owned by gridline-session

- **WHEN** the owning package of the registered Gridline session entry is queried
- **THEN** the answer is `gridline-session`

### Requirement: gridline-config owns defaults

`gridline-config` SHALL own the shipped defaults under `/etc/xdg`, the default keybindings, and the default themes.

#### Scenario: Defaults are owned by gridline-config

- **WHEN** the owning package of a shipped default under `/etc/xdg` is queried
- **THEN** the answer is `gridline-config`

#### Scenario: Local changes survive upgrade

- **WHEN** an administrator has modified a shipped default and `gridline-config` is upgraded
- **THEN** the local modification is preserved rather than silently overwritten

### Requirement: gridline-tools owns the command-line surface

`gridline-tools` SHALL own `gridline doctor` and the theme and diagnostic commands.

#### Scenario: Commands are owned by gridline-tools

- **WHEN** the owning package of the `gridline` command is queried
- **THEN** the answer is `gridline-tools`

#### Scenario: Tools are independently installable

- **WHEN** `gridline-tools` is installed without a Gridline session running
- **THEN** its commands run and report on the system rather than failing to start

### Requirement: gridline-desktop is reserved as a metapackage boundary

The name `gridline-desktop` SHALL be reserved for a future metapackage that pulls in the packages making up a complete Gridline desktop, and SHALL NOT be delivered by this change. When delivered, it SHALL contain no functionality of its own.

#### Scenario: Reserved name is not delivered prematurely

- **WHEN** the output of this change is inspected
- **THEN** no `gridline-desktop` package is produced, and the reserved boundary is recorded in the project documentation

#### Scenario: Metapackage ships no implementation when delivered

- **WHEN** a future change delivers `gridline-desktop` and it is inspected
- **THEN** it ships no executable, configuration, or data files beyond its own packaging metadata

### Requirement: Images consume packages and add no core functionality

Any future Ubuntu Remix or ISO SHALL obtain Gridline solely by installing these packages, and SHALL NOT be the only place a core Gridline capability exists.

#### Scenario: Image offers nothing exclusive

- **WHEN** a Gridline capability is present in a future image
- **THEN** the same capability is available by installing the Gridline packages on a stock Ubuntu system

### Requirement: Gridline is removable

Removing the Gridline packages SHALL return the system to a working stock Ubuntu desktop.

#### Scenario: Clean removal

- **WHEN** all Gridline packages are removed
- **THEN** no Gridline session entry remains, the Ubuntu GNOME session works, and no Gridline-owned files are left outside of administrator-modified configuration
