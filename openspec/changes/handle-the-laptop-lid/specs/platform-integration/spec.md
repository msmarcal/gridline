# Spec Delta

## ADDED Requirements

### Requirement: A closed lid takes the laptop's own screen out of the session

A Gridline session SHALL stop using a laptop's built-in screen while its lid is closed and another screen is attached, and use it again when the lid opens. It SHALL NOT stop using it when no other screen is attached, and if the decision cannot be made or applied the built-in screen SHALL stay in use.

#### Scenario: A session that starts with the lid closed and a screen attached

- **WHEN** a person logs in to a Gridline session on a laptop whose lid is closed and which has an external screen attached
- **THEN** the session's bar, windows, pointer and keyboard focus are on the external screen, and nothing the person needs appears on the built-in screen

#### Scenario: The lid closes during a session

- **WHEN** the lid is closed during a Gridline session while an external screen is attached
- **THEN** the built-in screen is no longer part of the session, and the windows that were on it remain reachable on the external screen

#### Scenario: The lid opens

- **WHEN** the lid is opened during a Gridline session that had taken the built-in screen out
- **THEN** the built-in screen is part of the session again, at its own preferred mode

#### Scenario: Closing the lid with no other screen attached

- **WHEN** the lid is closed and no external screen is attached
- **THEN** the built-in screen is not taken out of the session, and the machine's own lid behavior, which is to suspend by default, decides what happens

#### Scenario: The lid is handled while the screen is locked

- **WHEN** the lid moves while the session is locked
- **THEN** the built-in screen is handled in the same way, and the lock stays in place

#### Scenario: A failure leaves the built-in screen in use

- **WHEN** the lid's state or the attached screens cannot be determined, or taking the built-in screen out fails
- **THEN** the built-in screen stays in use and the session is otherwise unaffected
