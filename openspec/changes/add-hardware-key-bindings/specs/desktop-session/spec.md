# Spec Delta

## ADDED Requirements

### Requirement: A session responds to the keys the machine already has

A Gridline session SHALL act on the hardware keys the machine ships with - at minimum volume up, volume down, mute, microphone mute, and the media keys for play or pause, previous and next - without the user configuring anything. Where the machine has a backlight, the brightness keys SHALL act on it.

#### Scenario: A volume key changes the volume

- **WHEN** a user presses the volume up or volume down key during a Gridline session
- **THEN** the session's audio output volume changes in that direction, and the system's own view of the volume reflects it rather than the change living only in whatever was asked

#### Scenario: Mute is a toggle and is reversible

- **WHEN** a user presses the mute key twice
- **THEN** the output is silenced and then restored to the level it had, rather than restored to a level the session chose

#### Scenario: The keys work while the screen is locked

- **WHEN** a hardware key is pressed while the session is locked
- **THEN** it acts, because silencing a machine or dimming a screen is not a privileged operation and a key that stops working at the lock screen is a key that works half the time

#### Scenario: Holding a key repeats it

- **WHEN** a volume or brightness key is held down
- **THEN** the value continues to change while it is held, rather than changing once per press

#### Scenario: A media key reaches whatever is playing

- **WHEN** a media key is pressed while an application is playing audio
- **THEN** that application receives it, whichever application it is, because the session addresses players through the interface they already advertise rather than knowing any of them by name

#### Scenario: Brightness is asked of the system, not taken

- **WHEN** a brightness key is pressed on a machine with a backlight
- **THEN** the backlight changes without the session requiring privileges it was not given, and without Gridline having altered group membership, udev rules, or any file outside the paths it owns

#### Scenario: The keymap names no tool

- **WHEN** the shipped compositor configuration is read to find what a hardware key runs
- **THEN** it names a Gridline-owned program rather than the utility that performs the action, so that replacing the utility is a change to one Gridline file and not to a user's keymap
