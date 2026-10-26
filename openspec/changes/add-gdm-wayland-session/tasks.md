# Tasks

Scope note: this change ships implementation. It creates the session entry and the launcher in `gridline-session` and verifies them on the development host, which is the declared baseline. No compositor is started, no adapter contract is defined, and no dependency on a compositor package is declared.

Verification note: the failure path can only be exercised from the login screen, so the tasks in group 4 require logging out of the current desktop. The expected outcome is a return to GDM with the stock session still selectable, which is itself the thing being verified.

## 1. Session entry

- [ ] 1.1 Create the session entry as a Gridline-named file installed into `/usr/share/wayland-sessions/`, with `Type=Application`, a `Name` and a `Comment`, and verify `dpkg-deb -c` on the built `gridline-session` lists the file and that no Gridline manifest declares the directory
- [ ] 1.2 Set `DesktopNames` in the entry so the identity resolves to Gridline first and the compositor second, and verify the semicolon-separated value in the file matches the colon-separated value the display manager is expected to produce
- [ ] 1.3 Set `Exec` to the launcher path and `TryExec` to the same path, and verify both point inside `/usr/libexec/gridline/`, which is the directory `gridline-session` already declares
- [ ] 1.4 Verify the entry declares no compositor package dependency and that `debian/control` for `gridline-session` gained no `Depends` entry, since the launcher reports a missing compositor rather than requiring one

## 2. Launcher

- [ ] 2.1 Create the launcher in `/usr/libexec/gridline/`, executable, installed by `gridline-session`, and verify `dpkg-deb -c` shows it with an executable mode and that the ownership gate still reports every path claimed once
- [ ] 2.2 Implement required-component discovery so the launcher establishes which components the session needs and identifies the first one missing, and verify running it directly on a host with no compositor adapter names the missing component on stderr and exits non-zero
- [ ] 2.3 Write the missing component to the system journal under an identifiable Gridline name, and verify `journalctl` can select the record by that identifier alone, without knowing the time it was written
- [ ] 2.4 Record the chosen journal identifier in `docs/architecture.md`, and verify it is stated as a contract that `add-session-health-check` depends on rather than as an implementation detail
- [ ] 2.5 Verify the launcher starts no compositor and attempts no fallback, by reading it for any invocation of a compositor binary and confirming there is none

## 3. Build and package verification

- [ ] 3.1 Build the packages and verify `dpkg-buildpackage` produces three `.deb` files and the ownership gate reports a file count matching what the packages contain
- [ ] 3.2 Install the packages and verify `dpkg -S` attributes the session entry and the launcher to `gridline-session`, and that `dpkg -S` on `/usr/share/wayland-sessions` still returns `ubuntu-session` among its owners rather than a Gridline package replacing it
- [ ] 3.3 Verify the stock `ubuntu.desktop` entry is untouched, by comparing its content and its owning package before and after installing Gridline
- [ ] 3.4 Remove the packages and verify the session entry and the launcher are both gone and `/usr/share/wayland-sessions/` still exists with the stock entry intact

## 4. Login screen verification on the baseline

- [ ] 4.1 With the packages installed, verify the Gridline session appears in GDM's session list alongside Ubuntu
- [ ] 4.2 Verify GDM still preselects the session the user last used rather than Gridline, by recording the preselected session before installing and after
- [ ] 4.3 Select the Gridline session and verify the login screen returns, then read the journal by the identifier from task 2.3 and confirm the missing compositor is named
- [ ] 4.4 From the stock session, verify the recovery procedure in `docs/recovery.md` locates that record using only tools present on a stock Ubuntu system, and correct the document if the procedure does not match what the launcher actually writes
- [ ] 4.5 Verify the Gridline entry is withheld when the launcher is absent, by removing the launcher file while leaving the entry installed and confirming the session is not offered while Ubuntu still is, then restoring it

## 5. Documentation and sequence

- [ ] 5.1 Record in `docs/architecture.md` that the desktop identity comes from the session entry rather than from the launcher, and verify the document states this supersedes the earlier decision and gives the reason
- [ ] 5.2 Record in `docs/component-ownership.md` that the session entry is a Gridline-named drop-in in a directory `ubuntu-session` owns, and verify the row states the fallback behavior for the entry being absent
- [ ] 5.3 Update `docs/package-matrix.md` so `gridline-session` lists the session entry and the launcher among what it owns, and verify the statement matches the ownership requirement in `openspec/specs/package-delivery/spec.md`
- [ ] 5.4 Verify every requirement added in `specs/desktop-session/spec.md` appears in this change's attribution table in `proposal.md` with exactly one owning change, and that no requirement is listed twice or omitted

## Workflow follow-up

- Run `/opsx:verify` once the entry and the launcher are built and the login screen checks have run.
- Archive this change, then propose `add-hyprland-session-baseline`, which replaces the missing-compositor failure with a real adapter.
