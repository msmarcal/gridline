# desktop-session Specification

## Purpose
Gridline's existence as a selectable Wayland desktop session on Ubuntu: how it registers with the display manager, how it starts and stops, how it coexists with the stock GNOME session that remains the user's supported fallback, and how a user recovers when a Gridline session fails.

## Requirements

### Requirement: Session is selectable in GDM

Gridline SHALL register a Wayland session entry that GDM presents as a selectable option on the login screen once `gridline-session` is installed.

#### Scenario: Session appears after install

- **WHEN** `gridline-session` is installed on Ubuntu 26.04 LTS x86_64 and GDM is restarted or the system is rebooted
- **THEN** GDM's session list offers a Gridline entry alongside the existing Ubuntu sessions

#### Scenario: Session disappears after removal

- **WHEN** `gridline-session` is removed
- **THEN** GDM no longer offers the Gridline entry, and no Gridline session entry remains registered

### Requirement: Session is opt-in, never the default

Installing Gridline SHALL NOT change which session GDM selects by default, for existing users or for users created after installation.

#### Scenario: Existing user's default is preserved

- **WHEN** a user whose last session was Ubuntu GNOME installs `gridline-session` and returns to the login screen
- **THEN** GDM still preselects Ubuntu GNOME, and Gridline is chosen only by explicit selection

#### Scenario: New user does not receive Gridline by default

- **WHEN** a user account is created on a system with Gridline already installed and that user reaches the login screen for the first time
- **THEN** GDM preselects the system's stock Ubuntu session rather than Gridline

### Requirement: GNOME session remains installed and supported

Installing or removing Gridline SHALL leave the regular Ubuntu GNOME session installed, selectable, and functional. Gridline is additive.

#### Scenario: GNOME survives Gridline install

- **WHEN** Gridline is installed
- **THEN** the Ubuntu GNOME session entry is still present in GDM and starts normally when selected

#### Scenario: GNOME survives Gridline removal

- **WHEN** Gridline is removed after use
- **THEN** the Ubuntu GNOME session starts normally, with no leftover Gridline state that alters its behavior

### Requirement: Session launch reaches a usable desktop or fails visibly

Selecting the Gridline session SHALL either reach a usable tiling desktop or return the user to the login screen with the failure recorded in the system journal.

The record SHALL be selectable by its author rather than by searching the journal's text, because a user reading it has just been returned to the login screen and knows neither which boot the failure belongs to nor how recently it happened.

#### Scenario: Successful launch

- **WHEN** a user selects the Gridline session and authenticates
- **THEN** a Wayland tiling desktop starts and accepts keyboard and pointer input

#### Scenario: Launch failure returns control to the user

- **WHEN** a required component of the session fails to start
- **THEN** the session exits, the user is returned to GDM, and the cause is written to the journal under the syslog identifier `gridline-session`

#### Scenario: The failure record is selectable without a time window

- **WHEN** a user returned to the login screen looks for the cause from the stock session
- **THEN** selecting records by the syslog identifier returns the launcher's records and no others, without naming a boot or a time range

### Requirement: A failed Gridline session is recoverable without Gridline

A user whose Gridline session cannot start SHALL be able to reach a working desktop and locate the failure record without needing a functioning Gridline session.

#### Scenario: Recovery to a working desktop

- **WHEN** the Gridline session fails to start and the user is returned to GDM
- **THEN** the Ubuntu GNOME session remains selectable from the same login screen and starts normally

#### Scenario: Failure is discoverable from the fallback session

- **WHEN** a user logs into the fallback session after a Gridline launch failure
- **THEN** the documented recovery procedure locates the Gridline failure record in the system journal using only tools present on a stock Ubuntu system

### Requirement: Session teardown releases the seat

Logging out of Gridline SHALL terminate the session's processes and return the seat to GDM.

#### Scenario: Clean logout

- **WHEN** a user logs out of a running Gridline session
- **THEN** session processes for that user terminate, and GDM presents the login screen again on the same seat

### Requirement: Session identifies itself to the desktop stack

The running session SHALL advertise a stable desktop identity so that portals, Snap confinement, and application defaults can resolve Gridline-specific behavior.

#### Scenario: Desktop identity is set

- **WHEN** a Gridline session is running
- **THEN** the session environment exposes a stable, documented desktop identity value that does not change between releases without a spec change

### Requirement: A session entry is offered only when its launcher is present

The Gridline session entry SHALL name the launcher it starts in a way the display manager can resolve before offering the session, so that an installation missing the launcher offers no session rather than one that cannot start.

The fallback is the stock session: when the Gridline entry is withheld, the login screen still offers Ubuntu, so a user is never left without a selectable desktop.

#### Scenario: Entry is offered when the launcher is present

- **WHEN** `gridline-session` is fully installed and the login screen is shown
- **THEN** the Gridline entry appears in the session list

#### Scenario: Entry is withheld when the launcher is absent

- **WHEN** the session entry is present but the launcher it names is not
- **THEN** the login screen does not offer the Gridline session, and the stock Ubuntu session is still offered and selectable

#### Scenario: A withheld entry is diagnosable

- **WHEN** the Gridline entry is expected but not offered
- **THEN** the launcher path the entry names can be read from the entry itself, so the missing file is identifiable without guessing

### Requirement: A session with a complete adapter reaches a usable desktop

When the active adapter reports every required capability, selecting Gridline SHALL reach a running tiling desktop that accepts input, rather than only failing visibly.

#### Scenario: The session starts

- **WHEN** a user selects Gridline and authenticates on a system whose adapter reports every required capability
- **THEN** a Wayland tiling desktop starts, accepts keyboard and pointer input, and the session is recorded as running for that user

#### Scenario: The compositor fails after the capability check passes

- **WHEN** every required capability is reported but the compositor fails to start or exits unexpectedly
- **THEN** the session ends rather than leaving a seat with no desktop, the cause is written to the journal under the session's syslog identifier, and the user is returned to the login screen with the stock Ubuntu session still selectable

#### Scenario: Logout releases the seat

- **WHEN** a user logs out of a running Gridline session
- **THEN** the compositor terminates, the seat returns to the display manager, and the login screen is usable without a reboot

### Requirement: The session refuses to start when a blocking component is absent

A session SHALL NOT start when a component the project records as blocking is absent, whether or not that component is reached through the compositor adapter, and the absent component SHALL be named in the failure record.

#### Scenario: A blocking component outside the adapter is absent

- **WHEN** a component the project records as blocking startup is not installed, and that component is not one the adapter reports on
- **THEN** the session does not start, the component is named in the journal under the session's syslog identifier, and the user is returned to the login screen with the stock Ubuntu session still selectable

#### Scenario: The check runs before the compositor

- **WHEN** a blocking component is absent
- **THEN** the launch ends before a compositor process is created, since nothing has to run for an absent file to be observed

#### Scenario: A capability is not a component

- **WHEN** the adapter reports the capability through which a blocking component is used, and that component is not installed
- **THEN** the session still refuses to start, because a compositor that implements a protocol is not the same thing as the component that speaks it

#### Scenario: Restarting units do not satisfy the rule

- **WHEN** a blocking component ships a service unit that restarts on failure
- **THEN** the session's own check decides whether to start, rather than treating a unit that will retry as evidence the component is present

### Requirement: A newly installed session is usable without configuration

A Gridline session SHALL be operable from the keyboard on first login, with Gridline's own defaults in effect, without the user having written any configuration. Configuration defaults are not a blocking component: their absence SHALL degrade the session rather than prevent it from starting.

Gridline SHALL deliver those defaults by a mechanism scoped to a Gridline session.

#### Scenario: The keyboard works on first login

- **WHEN** a user logs into Gridline for the first time on a machine where they have written no configuration of their own
- **THEN** the session is operable from the keyboard, including launching a terminal, closing a window, moving focus between windows and switching workspaces, without the user consulting the compositor's documentation to discover a binding

#### Scenario: Gridline's defaults are the ones in effect

- **WHEN** a Gridline session is running with no user configuration present
- **THEN** the configuration in effect is the one Gridline ships, distinguishable at runtime from the compositor's own autogenerated default, so that a reader can tell which of the two is being used rather than inferring it from the session appearing to work

#### Scenario: A first login presents no upstream project messaging

- **WHEN** a user logs into a freshly installed Gridline session
- **THEN** the desktop presents no banner, nag or warning raised by the components Gridline integrates, so that what the user sees on first login is the session rather than the state of the projects it is built from

#### Scenario: A suppressed warning is recorded rather than erased

- **WHEN** Gridline's defaults suppress a warning that reports a condition the project has not resolved
- **THEN** the condition remains recorded in the project's documentation with its owner, so that suppressing the message does not also remove the question from view

#### Scenario: A user's own configuration does not silently discard the defaults

- **WHEN** a user has written their own configuration for a component Gridline ships defaults for
- **THEN** their configuration is the one in effect, as it would be on a machine where Gridline had shipped none, and the session running without Gridline's defaults is recorded in a form a diagnostic can read rather than discovered by a key that does nothing

#### Scenario: The defaults reach no session but Gridline's

- **WHEN** a user starts a session of a component Gridline ships defaults for, other than through Gridline
- **THEN** that session resolves its configuration exactly as it would on a machine where Gridline were not installed, and none of Gridline's defaults applies to it

#### Scenario: Absent defaults degrade rather than block

- **WHEN** the package owning the shipped defaults is not installed, or its files have been removed
- **THEN** the session still starts and reaches a desktop, with the missing defaults reported as a capability gap, because a session without keybindings is a worse desktop and not an unsafe one

### Requirement: The login screen offers Gridline and not the compositor it is built on

Where a Gridline installation brings a compositor whose package registers its own session entry, the login screen SHALL offer Gridline and SHALL NOT also offer that compositor's entry, because the entry is present as a consequence of Gridline's dependency rather than a choice the user made. Removing Gridline SHALL restore it.

#### Scenario: One tiling session is offered

- **WHEN** Gridline is installed and the login screen lists the available sessions
- **THEN** Gridline is offered, the stock Ubuntu session is offered, and the compositor's own entry is not

#### Scenario: The compositor's entry returns when Gridline goes

- **WHEN** Gridline is removed from a machine where the compositor remains installed
- **THEN** the login screen offers the compositor's entry again, and the file it reads is the one the compositor's package shipped

#### Scenario: The fallback session is never hidden

- **WHEN** the mechanism that hides a session entry is applied
- **THEN** the stock Ubuntu session entry is not among those hidden, since every recovery path depends on being able to select it, and a machine whose only remaining entry is Gridline has no route back when Gridline does not start

#### Scenario: The compositor stays usable to someone who wants it

- **WHEN** a user wants the compositor's own session on a machine with Gridline installed
- **THEN** the compositor remains installed and startable, and restoring its entry is documented, since this hides an option rather than removing a capability

### Requirement: The session presents a desktop rather than a bare compositor

A Gridline session SHALL show the state of the machine, deliver the notifications applications send, and present a background, without the user installing or configuring anything. None of these components is blocking: each one's absence SHALL degrade the desktop rather than prevent the session from starting, because a desktop that shows less is worse and not unsafe.

#### Scenario: The state of the machine is visible

- **WHEN** a Gridline session is running
- **THEN** the user can see, without running a command, which workspace they are on, the time, and the state of the machine's power source and network

#### Scenario: A notification reaches the user

- **WHEN** an application sends a desktop notification during a Gridline session
- **THEN** the notification is shown to the user and then dismissed or expires, rather than being discarded because nothing is listening for it

#### Scenario: The desktop has a background

- **WHEN** a Gridline session reaches the desktop with no application window covering the screen
- **THEN** a background is displayed, rather than whatever the compositor leaves behind when nothing has been drawn

#### Scenario: Each component is kept running

- **WHEN** one of these components has reached a working state during a session and then exits, for any reason
- **THEN** it is started again without the user intervening, and the session continues through the gap rather than ending

#### Scenario: A component that cannot start is not held forever

- **WHEN** one of these components fails to start repeatedly, rather than exiting after having run
- **THEN** the session stops starting it again, and the condition is reported as a capability gap rather than left as a component that is restarted for as long as the session lasts

#### Scenario: An absent component degrades the desktop and does not block the session

- **WHEN** one of these components is not installed, or fails to start
- **THEN** the session still starts and reaches a usable desktop, with the remaining components unaffected, and the condition reported as a capability gap rather than refusing the session

#### Scenario: Their configuration is a shipped default a user can decline

- **WHEN** a user replaces the configuration of one of these components with their own
- **THEN** their configuration applies, exactly as it would on a machine where Gridline had shipped none, and the session does not reimpose Gridline's

#### Scenario: Only the Gridline session is affected

- **WHEN** a user logs into a session other than Gridline on a machine where Gridline is installed
- **THEN** none of these components is started for that session, and that session presents what it presented before Gridline was installed

### Requirement: An application can be started without a terminal

A Gridline session SHALL let a user find an installed application by name and start it, from the keyboard, without opening a terminal and without configuring anything. The component is not blocking and runs on demand.

#### Scenario: An installed application can be found and started

- **WHEN** a user asks for the application launcher during a Gridline session and types part of an installed application's name
- **THEN** that application is offered and selecting it starts it, without the user naming a path or a package

#### Scenario: Reaching it needs no pointer and no prior knowledge of a command

- **WHEN** a user has a Gridline session and a keyboard
- **THEN** the application launcher is reachable by a key combination the session ships, and that combination is documented in a file the user can read without the session running

#### Scenario: Not running is not a fault

- **WHEN** no application launcher is on screen during a healthy Gridline session
- **THEN** the session reports the component as installed and able to run rather than as stopped or failed, so that the usual state of an on-demand component does not read as a defect

#### Scenario: An absent launcher degrades the desktop and does not block the session

- **WHEN** the application launcher is not installed, or fails to start
- **THEN** the session still starts and reaches a usable desktop, and the condition is reported as a capability gap rather than refusing the session

#### Scenario: The session's binding does not name the launcher it ships

- **WHEN** the shipped compositor configuration is read to find what the application launcher binding runs
- **THEN** it names a Gridline-owned program rather than the launcher's own executable, so that replacing the launcher is a change to one Gridline file and not to a user's keybindings

#### Scenario: Its configuration is a shipped default a user can decline

- **WHEN** a user replaces the application launcher's configuration with their own
- **THEN** their configuration applies, exactly as it would on a machine where Gridline had shipped none, and the session does not reimpose Gridline's

#### Scenario: Only the Gridline session is affected

- **WHEN** a user logs into a session other than Gridline on a machine where Gridline is installed
- **THEN** nothing Gridline ships for the application launcher changes what that session does, and that session presents what it presented before Gridline was installed

### Requirement: The session's own actions are reachable without knowing their commands

A Gridline session SHALL present its own actions - locking the screen, ending the session, suspending, restarting and powering off - so that a user can perform each one without knowing a key combination, a command, or which program implements it. The component is not blocking and runs on demand.

#### Scenario: Each action is listed and can be chosen

- **WHEN** a user asks for the command menu during a Gridline session
- **THEN** locking, ending the session, suspending, restarting and powering off are each listed in words that say what they do, and choosing one performs it

#### Scenario: An action asked for here is the same action asked for elsewhere

- **WHEN** the command menu's lock entry is chosen
- **THEN** the session is locked by the same path a lock requested from the keyboard, the idle timeout or another program takes, rather than by the menu starting a locker of its own, because `session-manageability` requires lock policy to be one policy and not several components configured independently

#### Scenario: Ending the session releases the seat

- **WHEN** the command menu's entry for ending the session is chosen
- **THEN** the session ends and the seat is returned to the display manager, which is the behavior `desktop-session` already requires of session teardown however it was triggered

#### Scenario: An action that cannot be performed says so

- **WHEN** an action in the menu is chosen and the system refuses it, or the component that performs it is unavailable
- **THEN** the user is told that it did not happen, rather than the menu closing and leaving them to infer the outcome from the machine's behavior

#### Scenario: Losing it costs no action

- **WHEN** the command menu is not installed or fails to start
- **THEN** every action it offers remains available by the means that existed before it - a key combination or a command - and the session still starts, with the condition reported as a capability gap

#### Scenario: Only the Gridline session is affected

- **WHEN** a user logs into a session other than Gridline on a machine where Gridline is installed
- **THEN** nothing Gridline ships for the command menu changes what that session does

### Requirement: A person can reach the machine's settings from the launcher

A Gridline session SHALL let a person open the machine's settings, including its network and Bluetooth panels, by choosing them in the application launcher, without a terminal and without configuring anything.

#### Scenario: Choosing a settings entry opens it

- **WHEN** a person finds and chooses Settings, Wi-Fi, Bluetooth or Network in the application launcher during a Gridline session
- **THEN** the corresponding panel appears, rather than the launcher closing and nothing opening

#### Scenario: The entries are marked for Gridline sessions only

- **WHEN** a launcher that honors the desktop entry's `OnlyShowIn` key is asked, in a Gridline session and in a session of another desktop, which of Wi-Fi, Bluetooth and Network to show
- **THEN** it shows them in the first and none of them in the second

#### Scenario: The session's identity is not changed to make it work

- **WHEN** the settings application is started from the launcher
- **THEN** only that program is given an identity it accepts, and every other program started from the launcher, the launcher itself and the session still name Gridline, because other programs and the desktop's portals choose their behavior by that identity

#### Scenario: A panel that cannot start says so

- **WHEN** the settings application fails to start from the launcher
- **THEN** the person is told that it did not open, rather than being left to work out whether the launcher or the machine is at fault

#### Scenario: Another session is unaffected

- **WHEN** a person is in a session other than Gridline's on a machine where Gridline is installed
- **THEN** the settings application starts exactly as it did before Gridline was installed

#### Scenario: Absence degrades and does not block

- **WHEN** the settings application is not installed
- **THEN** the session still starts and the launcher still works, the absence is not reported as a failure of Gridline, and every means of reaching the network and Bluetooth that existed before remains available
