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

### Requirement: An adapter answers a capability query in a defined form

The session SHALL be able to ask an adapter which capabilities it provides, and the adapter SHALL answer in a form the session can parse without knowing which compositor it is talking to.

#### Scenario: The query is answered

- **WHEN** the session asks an installed adapter what it provides
- **THEN** the adapter returns the set of capabilities it can deliver, in the defined form, and exits indicating whether the answer is complete

#### Scenario: The answer is compositor-neutral

- **WHEN** the session reads an adapter's answer
- **THEN** it determines the available capabilities without parsing compositor-specific output, so a second adapter needs no change in the session

#### Scenario: An unreported capability is absent

- **WHEN** an adapter's answer does not mention a capability
- **THEN** the session treats that capability as unavailable rather than assuming it, and silence is never read as availability

#### Scenario: An adapter that cannot answer

- **WHEN** an adapter is present but fails to answer, answers in a form the session cannot parse, or exits indicating failure
- **THEN** the session treats every capability as unreported and does not start, rather than proceeding on a partial answer

### Requirement: The session does not start when a required capability is unreported

A session SHALL NOT start unless the active adapter reports every capability the project records as required, and the first missing capability SHALL be named in the failure record.

#### Scenario: A required capability is missing

- **WHEN** the adapter answers without a capability the project records as required
- **THEN** the session does not start, the missing capability is named in the journal under the session's syslog identifier, and the user is returned to the login screen with the stock Ubuntu session still selectable

#### Scenario: A capability whose absence is a security exposure

- **WHEN** the unreported capability is one that exists to keep an unattended session from being left unlocked
- **THEN** the session still refuses to start rather than running without it, because a desktop that cannot be shown to lock must not run

#### Scenario: An optional capability is missing

- **WHEN** the adapter answers without a capability the project records as optional
- **THEN** the session starts, and the unavailable capability is reported as a capability gap rather than silently absent

### Requirement: Adapter isolation is enforced at build time

The build SHALL fail when compositor-specific content is found outside the documented adapter path, and the decision SHALL rest on the path alone.

#### Scenario: A leak outside the adapter path

- **WHEN** a file outside the documented adapter path carries a compositor binary name, socket or protocol name, configuration format, or process name
- **THEN** the build fails and names the file and the reference

#### Scenario: The check does not judge intent

- **WHEN** the check classifies a file
- **THEN** it uses the documented path as its only input, so a file is adapter content because of where it is and not because a reviewer judged it compositor-specific
