# Spec Delta

## ADDED Requirements

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
