# Tasks

Scope note: this change ships implementation. It creates the session entry and the launcher in `gridline-session` and verifies them on the development host, which is the declared baseline. No compositor is started, no adapter contract is defined, and no dependency on a compositor package is declared.

Verification note: the failure path can only be exercised from the login screen, so the tasks in group 4 are performed at the greeter rather than from a terminal. The expected outcome is a return to GDM with the stock session still selectable, which is itself the thing being verified. Group 4 carries an execution note describing how to reach the greeter without ending the running session.

## 1. Session entry

- [x] 1.1 Create the session entry as a Gridline-named file installed into `/usr/share/wayland-sessions/`, with `Type=Application`, a `Name` and a `Comment`, and verify `dpkg-deb -c` on the built `gridline-session` lists the file and that no Gridline manifest declares the directory
- [x] 1.2 Set `DesktopNames` in the entry so the identity resolves to Gridline first and the compositor second, and verify the semicolon-separated value in the file matches the colon-separated value the display manager is expected to produce
- [x] 1.3 Set `Exec` to the launcher path and `TryExec` to the same path, and verify both point inside `/usr/libexec/gridline/`, which is the directory `gridline-session` already declares
- [x] 1.4 Verify the entry declares no compositor package dependency and that `debian/control` for `gridline-session` gained no `Depends` entry, since the launcher reports a missing compositor rather than requiring one

## 2. Launcher

- [x] 2.1 Create the launcher in `/usr/libexec/gridline/`, executable, installed by `gridline-session`, and verify `dpkg-deb -c` shows it with an executable mode and that the ownership gate still reports every path claimed once
- [x] 2.2 Implement required-component discovery so the launcher establishes which components the session needs and identifies the first one missing, and verify running it directly on a host with no compositor adapter names the missing component on stderr and exits non-zero
- [x] 2.3 Write the missing component to the system journal under an identifiable Gridline name, and verify `journalctl` can select the record by that identifier alone, without knowing the time it was written
- [x] 2.4 Record the chosen journal identifier in `docs/architecture.md`, and verify it is stated as a contract that `add-session-health-check` depends on rather than as an implementation detail
- [x] 2.5 Verify the launcher starts no compositor and attempts no fallback, by reading it for any invocation of a compositor binary and confirming there is none

## 3. Build and package verification

- [x] 3.1 Build the packages and verify `dpkg-buildpackage` produces three `.deb` files and the ownership gate reports a file count matching what the packages contain
- [x] 3.2 Install the packages and verify `dpkg -S` attributes the session entry and the launcher to `gridline-session`, and that `dpkg -S` on `/usr/share/wayland-sessions` still returns `ubuntu-session` among its owners rather than a Gridline package replacing it
- [x] 3.3 Verify the stock `ubuntu.desktop` entry is untouched, by comparing its content and its owning package before and after installing Gridline
- [x] 3.4 Remove the packages and verify the session entry and the launcher are both gone and `/usr/share/wayland-sessions/` still exists with the stock entry intact

## 4. Login screen verification on the baseline

Execution note: these verifications do not require ending the current session.
Use Switch User to reach the greeter on a second VT, which leaves the running
session intact on its own VT. Selecting Gridline there exercises the failure
path, and returning to the original VT recovers a session that was never
closed. Verifying with a second account rather than the account already logged
in keeps the session selection real, because a greeter may offer to resume an
existing session instead of starting one. The residual risk is a launcher that
hangs instead of exiting, which leaves its VT occupied; the recovery is
switching back to the original VT, so run this at the physical console rather
than over SSH.

- [x] 4.1 Record the session the greeter preselects for the existing account, then install the
  packages, and verify the recording was taken before installation so task 4.4 has a
  before value to compare against
- [x] 4.2 Create a local test account with no administrative privileges **after** the packages
  are installed, used only for the greeter verifications in this group, and verify it has a
  valid login shell and appears in the greeter user list. The order matters: the second
  scenario of the opt-in requirement is a user created on a system where Gridline is already
  installed, so an account created before installation cannot exercise it
- [x] 4.3 With the packages installed, verify the Gridline session appears in GDM's session list alongside Ubuntu
- [ ] 4.4 Verify installing Gridline does not change what GDM preselects, by recording the
  preselected session at the greeter before installing and again after, and confirming the
  two match. Note that AccountsService writes `Session=` only once a user picks a session
  other than the default, so on this baseline the recorded value is the system default
  rather than a remembered choice: the invariant is that it is unchanged and is not
  Gridline, not that it names a previously used session. Check it for both accounts: the
  existing one against the value recorded in task 4.1, and the test account created after
  installation, whose first visit to the greeter is the second scenario of the requirement
- [x] 4.5 Select the Gridline session and verify the login screen returns, then read the journal by the identifier from task 2.3 and confirm the missing compositor is named
- [x] 4.6 From the stock session, run the recovery procedure exactly as `docs/recovery.md`
  currently words it and capture what it returns, then run the selection by journal
  identifier and capture that, before changing anything. The difference between the two
  outputs is the record of why the correction exists, and running them in the other order
  loses it. Then verify the procedure locates the record using only tools present on a
  stock Ubuntu system, and correct the document if
  the procedure does not match what the launcher actually writes. Note that the phrase
  "under an identifiable Gridline unit or process name" is not merely vague: the launcher
  is started by the display manager and has no unit of its own, which is the argument
  `docs/architecture.md` records for choosing a syslog identifier, so the sentence
  describes a mechanism the project has already documented as unavailable. Its replacement
  must name the mechanism as a syslog identifier, because that is what `journalctl -t`
  selects; a sentence that stays generic reproduces the same defect in new words. Then
  verify the identifier is spelled identically in
  `docs/recovery.md`, in `docs/architecture.md` where the contract is declared, and in the
  launcher itself, and add `docs/recovery.md` to the readers named alongside
  `add-session-health-check`: a contract with two readers and two spellings is not a
  contract. Record whether the document needed correcting or already matched, since the
  checkbox alone cannot distinguish the two outcomes
- [ ] 4.7 Verify the Gridline entry is withheld when the launcher is absent, by removing the
  launcher file while leaving the entry installed and confirming the session is not offered
  while Ubuntu still is, then restoring it and verifying both that `dpkg --verify
  gridline-session` reports nothing and that the launcher's mode is `755` and it executes.
  Both checks are needed: `dpkg --verify` compares md5 only, so a file restored with the
  wrong permissions passes it while the launcher cannot run, which is the failure that
  makes the entry resolve and the session die on selection. Reinstalling the same `.deb`
  is the method that restores content and mode together. Copying the source file back
  restores whatever mode the checkout produced, which is `0777 & ~umask` for a file git
  records as executable, so it matches the `755` that `dh_fixperms` produces only when the
  clone was made with `umask 022`. A divergence that depends on the cloner's umask is
  worse than a constant one, because it survives review on whichever machine happens to
  have the favourable value

- [ ] 4.8 Remove the test account and its home directory, and verify no human
  account other than the baseline owner remains and that no files under the
  removed home directory survive
## 5. Documentation and sequence

- [x] 5.1 Record in `docs/architecture.md` that the desktop identity comes from the session entry rather than from the launcher, and verify the document states this supersedes the earlier decision and gives the reason
- [x] 5.2 Record in `docs/component-ownership.md` that the session entry is a Gridline-named drop-in in a directory `ubuntu-session` owns, and verify the row states the fallback behavior for the entry being absent
- [x] 5.3 Update `docs/package-matrix.md` so `gridline-session` lists the session entry and the launcher among what it owns, and verify the statement matches the ownership requirement in `openspec/specs/package-delivery/spec.md`
- [x] 5.4 Verify every requirement added in `specs/desktop-session/spec.md` appears in this change's attribution table in `proposal.md` with exactly one owning change, and that no requirement is listed twice or omitted
- [x] 5.5 Reorder the downstream sequence table in `docs/architecture.md` so the recorded order matches the order actually executed, placing this change before `define-supported-ubuntu-integration`, and verify every change already archived appears above every change not yet started
- [x] 5.6 Record in `docs/architecture.md` why the inversion is safe, namely that this change declares no dependency entry and so needs nothing from the component matrix `define-supported-ubuntu-integration` produces, and verify the claim against the result of task 1.4

## Workflow follow-up

- Run `/opsx:verify` once the entry and the launcher are built and the login screen checks have run.
- Archive this change, then propose `add-hyprland-session-baseline`, which replaces the missing-compositor failure with a real adapter.
