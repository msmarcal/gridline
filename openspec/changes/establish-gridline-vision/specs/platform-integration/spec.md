# Spec Delta

## Purpose

The integration surface a Gridline session must satisfy to count as a usable Ubuntu desktop rather than a bare compositor: networking, audio, Bluetooth, desktop portals, Snap applications, and the Ubuntu security model.

## ADDED Requirements

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

### Requirement: Snap applications run correctly

A Gridline session SHALL support Snap applications, including those relying on confinement interfaces for display, audio, and portal access.

#### Scenario: Confined graphical Snap runs

- **WHEN** a user launches a strictly confined graphical Snap application in a Gridline session
- **THEN** the application displays, accepts input, and reaches audio and portal services through its confinement interfaces

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
