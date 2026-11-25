# Deck

**Status:** partly built; not tested on a booted system. The window runs against a real `foundationd` on a virtual display in the
unit tests (`foundation/tests/test_deck.py`, 27 tests); nothing has driven it on an installed system with a polkit agent, and
the job pages (install, add, remove) are covered only up to the dialogs.

Deck (`ersa-deck`, package `foundation-deck`) is the graphical client for managing the system, laid out like a settings app:
a sidebar of pages in groups (with a search field), one page at a time. At the foot of the sidebar a task panel shows what is
running, with progress and Cancel, and what has just finished. A banner appears on every page while an update waits for a restart,
and after a rollback (an update whose three start attempts were used up). A failed call is shown on its own page, not as a toast.
Shortcuts: Ctrl+F search, Ctrl+R or F5 refresh, Ctrl+T task panel, Ctrl+W or Ctrl+Q close. Below about 720 px the sidebar
and the page take turns. It holds no policy: every action is a `foundationd` D-Bus call that polkit
authorizes, like the `foundation` CLI. Python, GTK 4, libadwaita.

| Group | Page | Does | Calls |
| --- | --- | --- | --- |
| | Overview | the running version and whether this start was confirmed good; the system's facts (name, version, base, architecture, developer mode), the versions of Deck and `foundationd`, the project website | `Boot.GetBootState`, `Manager.GetSystemInfo`, `GetStatus`, `Version` |
| Software | Updates | check, install the newest, install from a file (signed or, on request, unsigned), restart, automatic-update switches | `Updates.*`, `Config` |
| | Bundles and Extensions | installed bundles with their packages, ready-made bundles (notice shown first), single RPMs as extensions after a compatibility report | `Compat.*` |
| | Repositories | list, turn on and off, refresh, add (shows the key's fingerprint first) and remove your own; find a package and add it to a bundle | `Repos.*` |
| | Application Sources | Flatpak sources: list, turn on and off, add (shows the key's fingerprint first), remove | `Apps.ListRemotes`, `InspectRemote`, `AddRemote`, `SetRemoteEnabled`, `RemoveRemote` |
| | Base Packages | searchable read-only list of what the image contains | `Manager.ListBasePackages` |
| System | Boot Slots | both slots, start the other once or always, empty a standby slot, restart | `Boot.*` |
| | Appearance | boot splash light or dark, shown as chosen | `Boot.GetSplash`, `SetSplash` |
| | Remote Access | SSH on or off, password sign-in, authorized keys | `Remote.*` |
| | Advanced | every registered setting, grouped, typed from the schema | `Config.*` |
| History | Activity | what the system did on request, with each job's log | `Manager.ListJobs`, `Job.Logs` |

Update-found notifications are not Deck's: `foundation-update-notify` (a user unit) sends them. Not in Deck: Flatpak applications (a store such as Bazaar, see [applications](applications.md)), network, display, users and the
desktop's own appearance (GNOME Settings).
`ersa-updater` was removed; Deck replaces it (the package `foundation-deck` obsoletes `foundation-updater`).

Tests: `ECP_MKOSI_DIR=$PWD python3 -m unittest discover -s foundation/tests -p test_deck.py`; the window tests need
`ERSA_GUI_TESTS=1` on a virtual display (see the top of that file).
