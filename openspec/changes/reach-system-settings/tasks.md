# Tasks

## 1. Before anything is built

Every earlier component here was measured before it was configured, and each time the measurement changed what got built. The workaround was only tried by hand, with a window opened and closed. The design rests on two claims that nobody has driven.

- [x] 1.1 Record the failure as a person meets it, with the session unlocked: open the launcher, type Wi-Fi, choose it, and read the compositor's client list and the program's own stderr. This is the baseline that task 3.1 is the opposite of
- [x] 1.2 Establish that the launcher hands its own `PATH` to what it starts: put a probe ahead on `PATH` for one launcher only, start an entry whose `Exec` names it, and confirm the probe ran and not the system's file. The whole design rests on this
- [x] 1.3 Measure what the real application does at launch: how fast it refuses, with what status, and that it stays up when it is accepted. Those numbers set the failure window and are not to be guessed
- [x] 1.4 Open every top-level panel of the settings application in turn with the workaround applied by hand, capture each, and classify it: opens with content, opens empty or with an error, or writes a setting nothing here applies. Record the list, and for each panel in the third class the process that would normally apply it and that it is not running

Findings, measured on the VM with the session unlocked:

- **1.1** Opening the real launcher and typing Wi-Fi shows an empty list, so nothing can be chosen. 28 of the 29 entries are `NoDisplay=true`; only "Settings" (`org.gnome.Settings`, `OnlyShowIn=GNOME;`) is listed. The baseline for Wi-Fi is therefore "not offered", not "offered and silent". This falsified the proposal's premise, which is why it is corrected there and the scope decision is open in design.md.
- **1.2** A probe on a PATH prefix given to one launcher ran when chosen from that launcher, and did not run from the control launcher. Programs started from the launcher inherit its PATH, so the stand-in design holds.
- **1.3** Refused with the Gridline identity: status 1 in about 150 to 180 ms. Accepted with `Gridline:Hyprland:GNOME`: the window appears in about 1.1 s and the process is alive 6 s later. A success never exits, so the failure window only has to be well above the 180 ms refusal; a few seconds is enough.
- **1.4** All 24 panels opened a window and stayed alive (`sharing` is not a panel of this version and falls back to Search). Classified from the captures, by reading them:
  - *Opens with content:* applications, background (Appearance), keyboard, network, notifications, online-accounts, privacy, search, sound, system, ubuntu (same as system), universal-access, wellbeing.
  - *Opens, and says why it is empty (no hardware in the VM):* bluetooth (No Bluetooth Found), wifi (No Wi-Fi Adapter Found), wwan, wacom, color (No Devices), printers (No Printers, and "some settings cannot be unlocked").
  - *Writes a setting nothing here applies (hypothesis, no applying process runs):* display (empty monitor preview, Night Light), power (Automatic Screen Blank), multitasking (Hot Corner, Window Resize), mouse (scroll direction, acceleration), background (accent color and dark style), keyboard (input sources). `gnome-shell`, `mutter` and the `gsd-*` daemons are not running (process list, with a known-good control). Whether each setting is applied by something else was not measured per panel.
  - Wi-Fi, Bluetooth and Network all open and are usable once reached; Network showed the VM's wired link with real addresses.

## 2. The stand-in and the entries

- [x] 2.1 Add the stand-in under `/usr/libexec/gridline/shims/` with the settings application's name. Verify it runs the real program by absolute path, appends `GNOME` to the existing identity rather than replacing it, and adds nothing else, by reading the running panel's environment from `/proc`
- [x] 2.2 Implement the failure window and the notification, with the constant and its reason at the top. Verify both directions: make the real path unusable and read that a notification names the failure, and start it normally and read that none appears
- [x] 2.3 Make `gridline-menu apps` put that directory first on `PATH` for the launcher it starts, in a Gridline session only. Verify from the launcher's own environment that it is there, and in a session with another identity that it is not
- [x] 2.4 Add the three entries under `src/applications/`, install them, and verify the launcher lists each in a Gridline session (typed query, screenshot) and that `desktop-file-validate` accepts them. Verify with a launcher that honors `OnlyShowIn` (GLib's `should_show`) that another identity lists none; fuzzel is not that launcher (1.1 follow-up, see design.md)
- [x] 2.5 Register the stand-in in the install manifest. Verify it is in the built package, executable, under a Gridline-named path, and that every build gate passes

Results (VM, built package installed):

- **2.1** The panel started through the stand-in has `XDG_CURRENT_DESKTOP=Gridline:Hyprland:GNOME` in `/proc/<pid>/environ`; the stand-in's own process keeps `Gridline:Hyprland`. Started with `GNOME` already present, it is not doubled. The one thing other than the identity that differs is the `PATH` the launcher gave it.
- **2.2** With the real program replaced by one that exits 3 at once: the stand-in exits 3 and one "Settings did not open" notification appears. With one that exits 3 after 6 s: no notification. Normal start: none. Control: a plain `notify-send` is seen by the same probe. The first run of this probe was wrong, counting mako's history only, which holds a notification only after it is dismissed, and gave the two cases the wrong way round; fixed to count active and dismissed. The real program was restored and `dpkg -V` is clean.
- **2.3** The launcher's `PATH` begins with `/usr/libexec/gridline/shims` under `Gridline:Hyprland` and does not under `GNOME`.
- **2.4** Entries installed and listed by the launcher. `desktop-file-validate` rejects `OnlyShowIn=Gridline;` (unregistered, wants `X-`); accepted and recorded in design.md. GLib shows them under `Gridline:Hyprland` and hides them under `GNOME`, with the archive's own entries behaving the other way, so the probe distinguishes. **fuzzel ignores the key**, so `gridline-menu apps` run by hand under another identity still lists them.
- **2.5** `dpkg-deb -c` shows the stand-in executable under `/usr/libexec/gridline/shims/` and the three entries under `/usr/share/applications/gridline-settings-*`; all six build gates pass.

## 3. Measured as a person reaches it

Needs the session unlocked, and a person to unlock it: the launcher takes keyboard focus and a locked session holds it.

- [x] 3.1 Drive the real launcher: open it, type Wi-Fi, press Return, and read which window the compositor lists. Do the same for Settings, Bluetooth and Network, and read that each window is the panel named. Each is the opposite of the 1.1 baseline
- [x] 3.2 Verify the identity is changed for nothing else: start another program from the same launcher and read its environment, then the launcher's, then the user manager's. All three must still name Gridline first and none may name GNOME
- [x] 3.3 Verify another session is unaffected: run the launcher role with another desktop's identity and confirm it adds nothing to `PATH`, so the settings application starts as it would have without Gridline
- [x] 3.4 Verify absence degrades: move the real program aside and choose a settings entry. The launcher must stay up, a notification must say it did not open, and the session must be unaffected

Results:

- **3.1** From the real launcher, typing each name and pressing Return: Settings opens the Network panel, Wi-Fi opens "No Wi-Fi Adapter Found" (no adapter in the VM), Bluetooth opens "No Bluetooth Found", Network opens the wired link. Read from the captures, because every panel's window class and title are `org.gnome.Settings` and `Settings`, so the compositor's client list cannot tell them apart. The launcher closed each time and no window remained after.
- **3.2** The launcher, a program started from it, the user manager and the compositor all read `XDG_CURRENT_DESKTOP=Gridline:Hyprland`; none names GNOME.
- **3.3** Under `XDG_CURRENT_DESKTOP=GNOME` the launcher role's `PATH` does not include the shims directory (task 2.3), so the real program is what a name resolves to.
- **3.4** With `/usr/bin/gnome-control-center` moved aside and Wi-Fi chosen: the launcher closed, one "Settings did not open" notification appeared, no window, the compositor stayed up, and `gridline doctor` lists the launcher as ready. The program was restored (`dpkg -V` clean).

## 4. Coherence

- [x] 4.1 Add settings to `docs/user-experience.md` and `docs/component-ownership.md`, which mention neither. State what works, what 1.4 found does nothing, and that typing the command in a terminal still fails because only the launcher is covered
- [x] 4.2 Add a check to the session-layer check that the stand-in's name still matches the program the real package ships. Verify it fails when the stand-in is renamed, and that where the application is not installed it names itself as unanswerable rather than passing
- [x] 4.3 Verify each scenario of the added requirement is exercised or recorded as unexercised with a reason, counting scenarios from the delta file rather than from the proposal's description of them

Results:

- **4.1** `docs/user-experience.md` has a Settings section and a row, with the panel classification and the limits (terminal, fuzzel and `OnlyShowIn`); `docs/component-ownership.md` has a row and the ownership list names the stand-in and entries.
- **4.2** The block was run on the VM in five states. As installed: silent. Stand-in named for a program with no declared owner: a problem. Stand-in named for a declared owner but the package no longer shipping `/usr/bin/<name>`: a problem. Stand-in running a different path: a problem. Owner package not installed: named as unanswerable. The block was exercised on its own, not through the whole check, because that check starts and stops units in the running user manager and the VM holds the live session; CI runs the whole of it.


- Run `/opsx:verify` once group 3 has been driven and group 1's panel classification is written down.
- If 1.4 finds that several panels do nothing, that is a follow-up change and not a reason to hold this one: reaching Wi-Fi and Bluetooth is the requirement, and the classification is what keeps the rest from being a surprise.
