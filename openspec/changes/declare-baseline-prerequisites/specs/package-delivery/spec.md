# Spec Delta

## ADDED Requirements

### Requirement: A package states what it assumes and does not declare

A Gridline package SHALL state, in the description an administrator reads before installing it, the conditions it needs to be useful that it does not express as dependencies. Stating an assumption is not declaring it: the package still installs, and the statement is what stops the installation being silent about it.

#### Scenario: The session says what it needs a machine to already have

- **WHEN** an administrator reads the description of the package that ships the session entry
- **THEN** it says that the entry is read by a display manager, that Gridline does not install or require one, and that on a machine without one the entry is inert rather than broken

#### Scenario: What the package ships is described, not promised

- **WHEN** a package's description is compared against the files it installs
- **THEN** the description names what is there rather than what is intended, so that a package which has gained content since the description was written is a defect the comparison finds

#### Scenario: An assumption that becomes a dependency stops being an assumption

- **WHEN** something a package stated as an assumption is later declared as a dependency
- **THEN** the statement is removed rather than left beside the declaration, because two places saying what a package needs is how one of them goes stale

### Requirement: A component reached through a mechanism is replaceable without removing Gridline

Where a Gridline session reaches a component through a freedesktop mechanism rather than by naming the program, that component SHALL be a recommendation rather than a dependency, so that replacing it is an ordinary package operation.

#### Scenario: The preferred component can be removed on its own

- **WHEN** a user removes the component Gridline prefers for a mechanism-reached role
- **THEN** the Gridline session package remains installed and the session still starts

#### Scenario: The preference still applies when nothing else expresses one

- **WHEN** a Gridline session reaches such a component and neither the user nor the machine has expressed a preference
- **THEN** Gridline's preferred component is the one used, in a Gridline session and in no other

#### Scenario: Removing the preferred component degrades to the mechanism's answer

- **WHEN** the component Gridline prefers is not installed
- **THEN** the session uses whatever the mechanism resolves on that machine, named by Gridline as a second preference rather than left to an unordered scan of every installed application

#### Scenario: A component Gridline configures by name stays a dependency

- **WHEN** a component is one the session hands a Gridline-owned configuration file naming that component
- **THEN** it is declared as a dependency, and this requirement does not make it a recommendation
