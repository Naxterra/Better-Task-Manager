# Fluent rebuild (WinUI 3)

Branch `fluent-ui`. The WinForms app in `src/BetterTaskManager` stays untouched until the new app reaches parity.

## Projects

| Project | Role |
|---|---|
| `src/BetterTaskManager.Core` | Collection only, no UI: one `NtQuerySystemInformation` call for all processes (private working set, CPU, I/O, handles), window/service lookups, PDH memory counters, native TCP/UDP tables, firewall rules. `MonitorEngine` produces one immutable snapshot per interval on a background thread. |
| `src/BetterTaskManager.Fluent` | WinUI 3, unpackaged, self-contained Windows App SDK 2.2 (same setup as DiskLoom). Mica, TitleBar search, NavigationView. |

## Design decisions

- **Memory = private working set**, the figure Task Manager's Memory column shows. The Performance page splits Task Manager's "In use" into apps, kernel pools, file cache, driver code and shared/other so the numbers reconcile.
- **Rows are reusable slots** (`SlotCollection`): each refresh writes values into existing rows instead of rebuilding the list, so scroll position, selection (tracked by row key) and container reuse survive every tick.
- **No UI work while minimized** (`MonitorHost.ViewSuspended`); chart history keeps recording.
- **Firewall rule names are unchanged** (`BetterTaskManager Block <sha1-12>`), so rules from the WinForms app are recognised. Standard users get one UAC prompt per change via the exe's `--firewall-block/--firewall-unblock` helper mode.
- **End task** matches Task Manager (no prompt for normal apps), verifies the PID's creation time before killing, warns for critical processes, and makes tree-kill an explicit separate action.

## Measured (2026-09-27, 20 logical CPUs, ~300 processes)

- Process read: 3.7 ms; full snapshot incl. network tables: ~27 ms.
- App at 1 s refresh: ~2.7% of one core with the Processes page visible, ~3% minimized.

## Next

1. Per-app network throughput (Mbit/s) and DNS names via ETW (`Microsoft-Windows-Kernel-Network`, `Microsoft-Windows-DNS-Client`); requires elevation.
2. Background service so connection history keeps recording while the window is closed; port `NetworkHistoryStore`.
3. German localization (resource-based, UI strings only — never user data).
4. Details view (all columns, per-process user), startup apps, efficiency mode.
5. Installer/CI switch to the Fluent app, then retire the WinForms project.
