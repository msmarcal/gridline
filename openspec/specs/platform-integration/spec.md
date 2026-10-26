# platform-integration Specification

## Purpose
The integration surface a Gridline session must satisfy to count as a usable Ubuntu desktop rather than a bare compositor: networking, audio, Bluetooth, desktop portals, Snap applications, power management, session locking, privileged action authorization, X11 compatibility, and the Ubuntu security model.

## Requirements

### Requirement: Networking is managed through NetworkManager

A Gridline session SHALL use NetworkManager for network state and SHALL NOT require the user to configure networking outside the session.

#### Scenario: Network state is visible and actionable

- **WHEN** a user is in a Gridline session on a machine with wired or wireless hardware
- **THEN** current connection state is observable from within the session, and connecting to an available network is possible without leaving it

#### Scenario: Secrets prompts are answerable

- **WHEN** NetworkManager requires a secret such as a Wi-Fi passphrase
- **THEN** the session presents a prompt that can accept the secret and completes the connection

### Requirement: Audio is served by PipeWire and WirePlumber

A Gridline session SHALL use PipeWire with WirePlumber as its session manager for audio, and SHALL NOT require manual audio setup.

#### Scenario: Audio works on login

- **WHEN** a user logs into Gridline on a machine with working audio hardware
- **THEN** playback and capture are available to applications, and the default output and input devices are selected without user intervention

#### Scenario: Output switching takes effect

- **WHEN** the user changes the default audio output device
- **THEN** subsequent playback uses the newly selected device

### Requirement: Bluetooth is managed through BlueZ

A Gridline session SHALL use BlueZ for Bluetooth, supporting pairing and connection of devices from within the session.

#### Scenario: Device pairing

- **WHEN** a user pairs a Bluetooth audio device or input device from within a Gridline session
- **THEN** the device pairs, connects, and is usable, with audio devices becoming available through PipeWire

### Requirement: Desktop portals are available with required backends

A Gridline session SHALL provide XDG Desktop Portals with a compatible Hyprland backend and the GTK backend, so that sandboxed and toolkit-based applications can reach portal-mediated features.

#### Scenario: Screen sharing works

- **WHEN** an application requests screen capture through the portal interface
- **THEN** the user is prompted to choose what to share, and the selected content is delivered to the application

#### Scenario: File chooser works

- **WHEN** an application requests a file through the portal file chooser
- **THEN** a file chooser is presented and the chosen file is delivered to the application

#### Scenario: Backend selection is deterministic

- **WHEN** more than one installed backend implements the same portal interface
- **THEN** the session resolves a documented, deterministic choice of backend rather than depending on start order

### Requirement: Portal backend selection is driven by a Gridline-owned configuration file

Gridline SHALL ship its own portal configuration file naming the backend preference for each required interface. It SHALL NOT rely on deprecated matching in a third-party portal definition, and SHALL NOT modify or replace a configuration file owned by another package.

#### Scenario: Backend selection follows Gridline's desktop identity

- **WHEN** the portal frontend resolves a backend and Gridline's desktop identity names Gridline first
- **THEN** the configuration file it reads is the one Gridline owns, selected by Gridline's own desktop name rather than by a name belonging to the compositor package

#### Scenario: Gridline's own portal configuration is the one that applies

- **WHEN** a Gridline session starts with the compositor portal backend and the GTK backend installed
- **THEN** the portal frontend resolves backends from the Gridline-owned configuration file selected by Gridline's desktop identity, and not from a file shipped by the compositor package

#### Scenario: Screen capture does not depend on fallback matching

- **WHEN** an application requests screen capture in a Gridline session
- **THEN** the compositor backend serves the request because the Gridline-owned configuration names it, without the frontend resorting to deprecated-key matching or last-resort fallback behavior

#### Scenario: A missing portal configuration is detected rather than silent

- **WHEN** Gridline's portal configuration file is absent or does not name a backend for a required interface
- **THEN** the condition is reported by the diagnostic commands as a configuration fault, rather than surfacing later as a portal request that fails with every component installed

#### Scenario: A missing Gridline configuration is not masked by a compositor default

- **WHEN** Gridline's portal configuration file is absent and the compositor package's own desktop name appears later in Gridline's desktop identity
- **THEN** the diagnostic commands still report the missing Gridline configuration as a fault, even though the portal frontend would resolve a working backend from the compositor package's configuration file

### Requirement: Snap applications run correctly

A Gridline session SHALL support Snap applications, including those relying on confinement interfaces for display, audio, and portal access.

#### Scenario: Confined graphical Snap runs

- **WHEN** a user launches a strictly confined graphical Snap application in a Gridline session
- **THEN** the application displays, accepts input, and reaches audio and portal services through its confinement interfaces

### Requirement: Power management and suspend behave as on a stock Ubuntu desktop

A Gridline session SHALL support the system's power management behavior, including suspend, resume, and lid handling, without requiring configuration outside the session.

#### Scenario: Suspend and resume

- **WHEN** a machine running a Gridline session suspends, whether by user request, idle timeout, or lid close
- **THEN** the system suspends, and on resume the session is still running with display, input, audio, and network restored

#### Scenario: Lid close on a laptop

- **WHEN** the lid is closed on a laptop running a Gridline session
- **THEN** the configured system behavior is honored, and an attached external display is handled without leaving the session without any usable output

#### Scenario: Battery state is observable

- **WHEN** a Gridline session runs on a machine with a battery
- **THEN** charge state and power source are observable from within the session

#### Scenario: Power events are attributable on failure

- **WHEN** suspend or resume fails
- **THEN** the failure is recorded in the system journal in a form the diagnostic commands can surface, and the session either remains usable or returns the user to GDM

### Requirement: The session is lockable and the lock survives suspend

A Gridline session SHALL provide screen locking, and a locked session SHALL require authentication before access is restored, including after suspend and resume.

#### Scenario: Manual lock requires authentication

- **WHEN** a user locks a running Gridline session
- **THEN** the session contents are no longer accessible, and authentication is required to return to the desktop

#### Scenario: Lock holds across suspend

- **WHEN** a locked Gridline session suspends and later resumes
- **THEN** the session is still locked and authentication is required

#### Scenario: Lock failure does not expose the session

- **WHEN** the lock component fails to start or crashes while the session is locked
- **THEN** the session is not left unlocked and unattended, and the failure is recorded in the system journal

### Requirement: Privileged actions can be authorized from within the session

A Gridline session SHALL provide a polkit authentication agent so that actions requiring authorization can be completed without leaving the session.

#### Scenario: Authorization prompt is answerable

- **WHEN** an application or system action requires polkit authorization during a Gridline session
- **THEN** the session presents a prompt that accepts credentials and completes or denies the action

#### Scenario: Missing agent is diagnosable

- **WHEN** no polkit authentication agent is available in the session
- **THEN** the condition is reported in a form the diagnostic commands can surface, rather than privileged actions failing without explanation

### Requirement: X11 applications run through XWayland

A Gridline session SHALL support X11 applications through XWayland, so that applications without native Wayland support remain usable.

#### Scenario: X11 application runs

- **WHEN** a user launches an application that supports only X11 in a Gridline session
- **THEN** the application displays, accepts keyboard and pointer input, and can use the clipboard with native Wayland applications

### Requirement: AppArmor confinement is preserved

Gridline SHALL NOT weaken or disable AppArmor confinement to make the session or its applications work.

#### Scenario: No profile is disabled for convenience

- **WHEN** Gridline is installed and a session is run
- **THEN** no existing AppArmor profile is disabled or set to complain mode by Gridline, and any profile Gridline ships is in enforce mode

### Requirement: Ubuntu security updates apply without reinstallation

Gridline SHALL receive security updates through the standard Ubuntu update mechanism, with no step that pins or holds the stack outside the archive.

#### Scenario: Updates arrive through APT

- **WHEN** a security update is published for Gridline or for a component it depends on
- **THEN** the standard update process installs it, and a subsequent Gridline session starts normally

### Requirement: The components a session requires are recorded with the behavior when each is absent

Gridline SHALL record, for each Ubuntu-provided component a session requirement depends on, whether that component's absence blocks the session from starting or degrades a capability within it.

#### Scenario: Every relied-on component appears in the record

- **WHEN** a session requirement depends on an Ubuntu-provided component
- **THEN** that component appears in the record, so no component is relied on informally

#### Scenario: Each entry states which of the two it is

- **WHEN** a component's entry is read
- **THEN** it states either that the component's absence blocks session startup or that it degrades a named capability, and not merely that the component is used

#### Scenario: A component that cannot be declared yet names its owner

- **WHEN** a component is recorded as blocking but no Gridline package can declare it, because the component is not yet selected or the session cannot yet use it
- **THEN** the record names the change that owns the declaration, so the gap is an attributed deferral rather than an omission

#### Scenario: The record and the eventual package relationship agree

- **WHEN** a Gridline package declares a relationship to a recorded component
- **THEN** a component recorded as blocking is expressed as a dependency and one recorded as degrading as a recommendation, so the record and the packaging cannot disagree silently

### Requirement: Each required component's archive support status is recorded

Gridline SHALL record the archive component that provides each required package on the supported baseline, because `main` carries a Canonical security-update commitment and `universe` does not, and that difference is a support fact rather than packaging detail.

#### Scenario: A universe component is identifiable as one

- **WHEN** a reader looks up what support a required component carries
- **THEN** the record names its archive component, and a component in `universe` is not presented as carrying a security-update commitment

#### Scenario: The gap is stated where the dependency is recorded

- **WHEN** Gridline material records that the session depends on a component in `universe`
- **THEN** the absence of a Canonical security-update commitment for that component is stated alongside the dependency, rather than only in a separate document a reader may not reach

### Requirement: Recorded component facts are verifiable against the supported archive

Each recorded fact about a required component SHALL be checkable against the supported baseline's archive, so that a record which has drifted is a detectable defect rather than a belief that survives because nothing tests it.

#### Scenario: A recorded fact is checked

- **WHEN** a recorded availability, version or archive component is compared against what the supported baseline's archive reports
- **THEN** the record matches, or the record is corrected

#### Scenario: The check can be repeated by a reader

- **WHEN** a reader wants to confirm a recorded fact
- **THEN** the records state how the measurement was taken and when, so repeating it needs no knowledge the records do not supply

#### Scenario: A component absent from the baseline

- **WHEN** a component the records name is not available on the supported baseline at all
- **THEN** the condition is treated as a defect in the record or in the component choice, and is not resolved by adding a repository outside the archive
