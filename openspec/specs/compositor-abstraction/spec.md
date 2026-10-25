# compositor-abstraction Specification

## Purpose
The boundary that keeps the compositor a replaceable component of Gridline rather than a dependency of the whole desktop, so that Hyprland can be the initial choice without the project becoming unable to change or add compositors later.

## Requirements

### Requirement: Compositor-specific logic is isolated

Compositor-specific logic SHALL be confined to an identifiable compositor adapter layer. Packages and components outside that layer SHALL NOT contain compositor-specific logic.

#### Scenario: Logic outside the adapter is compositor-neutral

- **WHEN** a component outside the documented adapter path is inspected
- **THEN** it contains no compositor-specific command invocations, socket protocols, configuration formats, or process assumptions

#### Scenario: Supporting a second compositor is confined

- **WHEN** support for an additional compositor is added in a later change
- **THEN** the work required outside the compositor adapter layer is limited to selecting which adapter is active

### Requirement: The adapter boundary is a single documented path

The compositor adapter layer SHALL occupy one documented filesystem path within the source tree, and that path SHALL be the boundary used to verify isolation.

#### Scenario: Boundary is documented and singular

- **WHEN** the project documentation is consulted for the adapter location
- **THEN** it names exactly one path as the adapter layer, and no compositor-specific component is shipped from outside it

#### Scenario: Isolation is mechanically checkable

- **WHEN** isolation is verified
- **THEN** the check distinguishes adapter content from non-adapter content by that documented path alone, without relying on reviewer judgment about which files are compositor-specific

### Requirement: No dependency on undocumented compositor internals

Gridline SHALL interact with the compositor only through interfaces the compositor documents as supported.

#### Scenario: Only documented interfaces are used

- **WHEN** the compositor adapter communicates with the compositor
- **THEN** it uses documented, supported interfaces, and does not parse internal state files, rely on undocumented socket commands, or depend on internal source layout

#### Scenario: Unsupported interface is rejected in review

- **WHEN** a proposed implementation requires an undocumented compositor behavior to work
- **THEN** it is treated as not meeting this requirement, and the need is recorded as an upstream gap rather than worked around silently

### Requirement: Compositor capabilities are discovered, not assumed

The session SHALL determine whether a required compositor capability is present rather than assuming it, and SHALL report a missing capability as a diagnosable condition.

#### Scenario: Missing capability is reported

- **WHEN** the session starts and a required compositor capability is unavailable
- **THEN** the condition is reported in a form the diagnostic commands can surface, naming the missing capability

### Requirement: Compositor identity is not exposed as Gridline's contract

Gridline's user-facing configuration and commands SHALL NOT require users to know which compositor is in use for behavior that is common across compositors.

#### Scenario: Common operations are compositor-neutral

- **WHEN** a user performs an operation Gridline defines for all compositors, such as applying a theme or running a diagnostic
- **THEN** the operation is expressed in Gridline's own terms and does not require compositor-specific arguments
