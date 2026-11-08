# Spec Delta

## ADDED Requirements

### Requirement: The project's own checks run without their author

The checks this project relies on SHALL run on a machine the author of a change does not control, and SHALL report a failure in a way that reaches whoever would otherwise merge it. A check that only runs when someone remembers to run it is a habit rather than a guard.

#### Scenario: A change that breaks a gate is reported without anyone running the gate

- **WHEN** a change is pushed that would fail one of the project's build-time checks
- **THEN** the failure is reported automatically, naming the check and what it found, without a person having chosen to run it

#### Scenario: What runs automatically and what a person runs are the same checks

- **WHEN** a contributor runs the project's checks on their own machine
- **THEN** they run the same checks the automated run performs, from the same definition, so that passing locally and failing afterwards means something changed rather than that two lists had drifted

#### Scenario: A check that cannot run automatically is named rather than omitted

- **WHEN** a verification the project relies on cannot run without a display, without hardware, or without a person looking at a screen
- **THEN** it is recorded as such with the reason, so that the set of things nobody is checking is visible rather than inferred from the set of things that are

#### Scenario: The automated run proves it can fail

- **WHEN** the automated verification is introduced or changed
- **THEN** it is shown failing against a deliberately broken input before it is relied upon, because a check that has only ever passed has not been shown to check anything
