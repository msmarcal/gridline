# session-manageability Specification

## Purpose
The mechanism by which an administrator sets Gridline session policy that a user cannot silently defeat, given that the session's components resolve configuration by first-match file lookup with no layering and no lock concept of their own.

## Requirements

### Requirement: Policy input is separate from the files components read

Gridline SHALL define a policy input owned by the administrator that is distinct from the configuration files session components read directly. Gridline SHALL NOT require an administrator to edit a component's own configuration file in order to set policy.

#### Scenario: Policy is expressed in one place

- **WHEN** an administrator sets a session policy that affects more than one component
- **THEN** it is expressed once in Gridline's policy input, rather than once per component in each component's own configuration format

#### Scenario: Component configuration is an output, not an input

- **WHEN** the configuration a component reads is inspected after policy has been applied
- **THEN** it is identifiable as generated from Gridline's policy input, and the documentation states that editing it directly is not the supported way to set policy

### Requirement: Effective configuration is produced by a Gridline-owned generator

Gridline SHALL own the mechanism that produces each component's effective configuration from the policy input and the user's own settings. The mechanism SHALL NOT depend on layering or merge behavior in a third-party configuration loader.

#### Scenario: Generation does not rely on search-path precedence

- **WHEN** the effective configuration for a component is produced
- **THEN** it is produced by Gridline's own mechanism, and its correctness does not depend on the component's configuration loader merging multiple files or honoring a directory order

#### Scenario: A component with first-match lookup is still governed

- **WHEN** a component resolves configuration by returning the first matching file and ignoring the rest
- **THEN** the file it finds is the one Gridline generated, and the policy input is reflected in it

#### Scenario: Generated output is traceable to its input

- **WHEN** an administrator or diagnostic inspects generated configuration
- **THEN** it identifies itself as generated and names the policy input it was produced from, so that an unexpected value can be traced without guessing

### Requirement: An administrator can lock an individual setting

Gridline SHALL allow an administrator to mark an individual setting as locked. A locked setting SHALL take effect in the session regardless of the user's own configuration, and SHALL NOT be defeated by the user supplying a configuration file of their own.

#### Scenario: A locked setting survives a user configuration file

- **WHEN** a setting is locked by policy and the user has written their own configuration for the same component
- **THEN** the locked setting is in effect in the session, and the user's value for that setting is not

#### Scenario: Unlocked settings remain user-configurable

- **WHEN** a setting is not locked by policy
- **THEN** the user's value for it takes effect, and locking one setting does not make the rest of that component's configuration read-only

#### Scenario: Locking is declared, not implied by file placement

- **WHEN** the set of locked settings is determined
- **THEN** it comes from an explicit declaration, and no setting is locked as a side effect of which directory a file was installed into

### Requirement: Policy that is not in effect is reported

Gridline SHALL detect when administrator policy is not in effect in a running session and SHALL report it as a fault. A superseded or unapplied policy SHALL NOT be indistinguishable from an applied one.

#### Scenario: Superseded policy is a reported fault

- **WHEN** a session runs with a policy setting that is not in effect, whether because generation did not run, the generated file was replaced, or a component read a different file
- **THEN** the diagnostic commands report the condition, naming the setting and the policy that was expected

#### Scenario: Policy state is inspectable without reproducing a failure

- **WHEN** an administrator asks whether policy is in effect on a machine
- **THEN** the answer is available from the diagnostic commands, without requiring the administrator to trigger the behavior the policy governs in order to observe it

#### Scenario: A locked setting that cannot be enforced is a fault, not a downgrade

- **WHEN** a setting is locked by policy and the session cannot enforce it
- **THEN** the condition is reported as a fault rather than silently falling back to the user's value

### Requirement: Screen lock policy is enforceable as a single policy

Gridline SHALL express screen lock policy, covering both when the session locks and what unlocking requires, as policy that an administrator can set and lock, even though the behavior is implemented by more than one component.

#### Scenario: Idle timeout and lock behavior are one policy

- **WHEN** an administrator sets screen lock policy
- **THEN** it is set once and governs both the idle timeout that triggers locking and the locking behavior itself, without the administrator configuring separate components independently

#### Scenario: Locked screen policy is not defeated by user configuration

- **WHEN** screen lock policy is locked and a user has their own configuration for either component involved
- **THEN** the session locks according to policy, and the diagnostic commands confirm the policy is in effect

### Requirement: Policy does not alter configuration resolution for other sessions

Gridline's policy mechanism SHALL NOT change configuration resolution for a session other than Gridline's, and removing Gridline SHALL leave no policy artifact that affects another session.

#### Scenario: A GNOME session is unaffected by Gridline policy

- **WHEN** Gridline is installed with policy set and a user logs into the stock GNOME session
- **THEN** that session resolves configuration exactly as it did before Gridline was installed, and no Gridline policy applies to it

#### Scenario: Removal leaves no policy residue

- **WHEN** Gridline is removed from a machine where policy was set
- **THEN** no generated configuration or lock declaration remains that changes the behavior of any remaining session
