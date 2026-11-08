# Spec Delta

## MODIFIED Requirements

### Requirement: A session component that is not running is reported

Gridline SHALL report the components a session expects but does not have. A user or administrator SHALL be able to find out which of them are running and which are absent or failed, without reading the journal and without knowing which unit implements which component.

#### Scenario: A missing component is nameable

- **WHEN** a session is running with one of its non-blocking components absent or failed
- **THEN** the diagnostic names that component, in the terms the project uses for it rather than the name of whatever package provides it, and says which of absent and failed it is

#### Scenario: A healthy session says so

- **WHEN** a session is running with every component it expects
- **THEN** the diagnostic reports that, rather than reporting nothing and leaving a reader to decide whether silence means healthy or means the diagnostic did not run

#### Scenario: The report distinguishes what refuses a session from what degrades it

- **WHEN** the diagnostic reports a component that is not running
- **THEN** it says whether that component's absence would have refused the session or degrades it, so that a reader is not left to infer severity from the component's name

#### Scenario: The diagnostic reads a record the session wrote

- **WHEN** a component is absent or fails during a session
- **THEN** the session records that, and the diagnostic reports from that record rather than from inspecting the state of units other packages own

#### Scenario: The diagnostic works when the session does not

- **WHEN** a session failed to start, or is not running at all
- **THEN** the diagnostic still runs and still reports, because the case a user most needs it in is the one where the desktop did not come up

#### Scenario: Reporting a gap never refuses a session

- **WHEN** a component is reported as a capability gap
- **THEN** the session's startup decision is unchanged by the reporting, and a component that is not blocking does not become blocking by being observable

#### Scenario: A component that stops after the session started is no longer reported as running

- **WHEN** a component the session expects is running when the session starts and then stops, fails or is given up on
- **THEN** the diagnostic stops reporting it as running, so that "running" is a statement about now rather than about when the session began

#### Scenario: The report says what it knows and not more

- **WHEN** the diagnostic reports a component whose state it has not been able to observe since it was first recorded
- **THEN** it says so rather than presenting a stale record as a current one, because a diagnostic that is confidently wrong is worse than one that admits a gap
