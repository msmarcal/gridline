# Proposal

## Why

Fourteen table cells across three live documents and three scenarios in `openspec/specs/` say a missing or failed session component is **reported as a capability gap**, and nothing on the machine reports one. The launcher's only reporting path, `report_missing`, ends in `exit 1` at every one of its eight call sites: it refuses a session and names the component, which is the blocking case and the opposite of this one. A user whose bar, notification daemon or wallpaper never started today learns it by noticing.

The same situation has a second half, decided while archiving `add-desktop-shell-components`: a component that cannot start at all is restarted forever. `Upholds=` revives a unit past its own start limit, measured at 59 starts in two minutes on a machine where the wallpaper daemon could not allocate buffers. A component that has given up should stop being held and should say so, which is why both halves belong to one change rather than two.

## What Changes

**This change ships implementation.**

- Adds `gridline doctor`, which reports the components a Gridline session expects, which of them are running, and which are absent or failed. `package-delivery` already names `gridline-tools` as its owner, so this delivers something the project owes rather than introducing a surface.
- Makes the session record a component that is absent or failed in a form the command reads, so the diagnostic surfaces a record rather than inferring state by inspecting units other packages own.
- Stops holding a component that cannot start, while keeping one that merely stops restartable. The distinction is the whole difficulty: `Upholds=` restarts a unit whatever state it reached, so neither removing `Restart=on-failure` nor the unit's own start limit bounds it, both measured.
- Corrects `docs/component-ownership.md`, which states the shortfall this change closes and overcounts it: it says seventeen cells across five live documents, and there are fourteen cells across three, with five further mentions in prose. The paragraph was written in the change that discovered the shortfall and counted occurrences rather than cells.

**Out of scope.** The launcher's blocking behaviour, which is a separate obligation and is correct: a session that cannot lock still must not start. The application launcher and the command menu, which are the next change. Theming. Nothing here changes which components are blocking.

## Capabilities

### New Capabilities

None.

### Modified Capabilities

- `session-manageability`: gains a requirement that the session's capability gaps are reported. The capability already requires that **policy** which is not in effect is reported, and this is the same shape for **components**: something the session expects is not true, and the administrator or user can find out without reading a journal. Putting it here rather than in `desktop-session` keeps the question of what the session does separate from the question of how its state is inspected.
- `desktop-session`: one requirement is modified. Its scenario about components being kept running says they are started again when they exit, for any reason, which is exactly what this change decides is wrong for a component that cannot start at all. The scenario has to distinguish the two rather than be quietly narrowed.

## Impact

- `packages/gridline/src/tools/`: the `gridline` command and its `doctor` subcommand, which do not exist yet; `gridline-tools` ships no executable today.
- `packages/gridline/debian/gridline-tools.install`: the command surface this package was created to own.
- `packages/gridline/src/session/`: where a component gap is recorded, and whatever expresses "stop holding this one".
- `packages/gridline/src/session/gridline-session.target`: the `Upholds=` entries, if bounding them turns out to belong in the target rather than in a drop-in.
- `docs/component-ownership.md`: the paragraph recording the shortfall, which this change closes and which also carries a wrong count.
- `docs/recovery.md`: how a user finds out what is missing, which today is "notice it".
- Verification runs on the LXD VM, where the wallpaper daemon genuinely cannot start, which is a real instance of the condition rather than one that has to be staged.
