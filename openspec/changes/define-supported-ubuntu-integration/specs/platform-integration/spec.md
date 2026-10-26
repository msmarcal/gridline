# Spec Delta

## ADDED Requirements

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
