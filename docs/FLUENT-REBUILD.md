# Better Task Manager — Fluent rebuild: handoff

Single source of truth for continuing this work in a new session. Read this before touching the code.

## 1. Goal and owner context

- Owner: Kaan (Naxterra). .NET developer; wants a partner, not a cheerleader; verify claims with live data.
- Goal: **replace Windows Task Manager** (numbers must reconcile with it) and get **Portmaster-style per-app network monitoring** without paying for Portmaster (Safing SPN not needed).
- UI must be **Fluent, not Excel-like, not laggy**. English UI for now; German localization later.
- GitHub (`github.com/Naxterra/Better-Task-Manager`) is only Kaan's **cloud backup**; nobody else uses it. Commit when asked; **do not push unless asked**.
- Kaan's sibling app **DiskLoom** (`D:\KI\Claude Code\DiskLoom`) is WinUI 3 / .NET 11 unpackaged; this rebuild copies its project setup and `Program.Main` pattern.

## 2. Repository state

- Local clone: `D:\KI\Codex\Windows Apps\Better-Task-Manager`.
- Branches: `main` = stale v1.0 WinForms; `codex/better-task-manager-preview-2` = WinForms v1.1-preview (Codex-built, in `src/BetterTaskManager`, still installed on Kaan's PC under `%LOCALAPPDATA%\Programs\Better Task Manager`); **`fluent-ui` = this rebuild** (branched from preview-2, local only, not pushed).
- Commits on `fluent-ui`:
  - `9a0a168` WinUI 3 rebuild + Core collector
  - `a1951e4` per-app network throughput (ETW) + Path column
  - `f3189f3` remote host names
- The WinForms project is untouched and still in the solution; retire it only after feature parity.

## 3. Build, run, test

```powershell
dotnet build BetterTaskManager.slnx -c Release          # whole solution, currently 0 warnings
dotnet build src/BetterTaskManager.Fluent -c Debug
src\BetterTaskManager.Fluent\bin\Debug\net11.0-windows10.0.26100.0\win-x64\BetterTaskManager.Fluent.exe --page Network
```

- `--page Processes|Performance|Network|Settings` opens a page directly (used for screenshots).
- SDK: .NET 11 RC (`11.0.100-rc.1`), Windows App SDK 2.2.0, Windows SDK BuildTools 10.0.28000.2270.
- Settings: `%LOCALAPPDATA%\BetterTaskManager\fluent-settings.json` (saved on graceful close only). Crash log: `fluent-crash.log` in the same folder.

### Testing pitfalls (all hit in practice)

- **Single instance**: a running copy makes new launches hand over and exit. A running copy also locks `bin\Debug`; build with `-o <scratch folder>` instead of killing Kaan's window.
- **Elevated copies** cannot be closed, clicked or captured from a non-elevated shell or computer-use (UIPI). Ask Kaan to close them.
- **computer-use grant**: request the lowercase basename `bettertaskmanager.fluent.exe`; the name "Better Task Manager" resolves to the installed WinForms app. The window may start hidden behind others: restore with `ShowWindow(h, 9)` + `SetForegroundWindow`.
- When the screen is busy (another session), capture the window with `PrintWindow(hwnd, dc, 2)` after `ShowWindow(h, 4)` (no focus steal).
- Admin-only features (ETW bandwidth, live DNS) were verified with a **separate elevated console harness** that references Core (`Start-Process -Verb RunAs`, writes results to a file); Kaan approves the UAC prompt. The harness lived in the session scratchpad; recreate it when needed (console exe, `ProjectReference` to Core, call `MonitorEngine.Start()`, set `Paused = true`, then drive `Collect()` once per second).
- The first click after a MenuFlyout closes is consumed by light-dismiss (WinUI behaviour, not a bug).
- In Git Bash, very long `python - <<'EOF'` heredocs with many quotes sometimes fail to parse; write patch scripts to a file and run them.

## 4. Architecture

```
src/BetterTaskManager.Core      (net11.0-windows10.0.26100.0, no UI)
  Native/NtProcessReader.cs     NtQuerySystemInformation(SystemProcessInformation) — all processes in one call
  Native/VisibleWindows.cs      one EnumWindows pass → PIDs with visible, uncloaked, titled top-level windows
  Native/ServiceNames.cs        EnumServicesStatusEx → PID → hosted service display names (read every 10 s)
  Native/MemoryCounters.cs      PDH via PdhAddEnglishCounter (locale-independent \Memory\* counters)
  Native/Win32.cs, NativeProcessInfo.cs   OpenProcess/QueryFullProcessImageName/IsProcessCritical/GetSystemTimes
  Network/NativeNetworkCollector.cs       GetExtendedTcp/UdpTable (copied from WinForms preview; offsets verified)
  Network/BandwidthMonitor.cs   kernel TCP/IP ETW (TraceEvent 3.2.6) → bytes per PID and per socket; flush before each sample
  Network/HostNameResolver.cs   IP → host name: DNS cache (WMI) + DNS-Client ETW (elevated) + PTR-only DnsQuery fallback
  Firewall/FirewallRules.cs     netsh outbound block rules; names "BetterTaskManager Block <SHA1(lower path)[..12]>" (compatible with WinForms app)
  Firewall/CommandRunner.cs     process runner with timeout (copied)
  Monitoring/MonitorEngine.cs   background loop → immutable MonitorSnapshot per interval
  Monitoring/Samples.cs         ProcessSample, ConnectionSample, MemoryBreakdown, SystemSample, MonitorSnapshot

src/BetterTaskManager.Fluent    (WinUI 3 unpackaged, self-contained WinAppSDK, DISABLE_XAML_GENERATED_MAIN)
  Program.cs                    custom Main: --firewall-block/--firewall-unblock helper mode, --wait-for-pid, single-instance mutex
  App.xaml(.cs)                 theme resources (heat/memory/chart brushes), styles; RestartElevated()
  MainWindow.xaml(.cs)          Mica, TitleBar with search box, NavigationView (LeftCompact), elevation InfoBar, Ctrl+1/2/3, Ctrl+F
  Services/MonitorHost.cs       owns MonitorEngine; coalesces snapshots to the UI thread; history buffers; search text;
                                firewall rule cache + SetBlockedAsync (elevated helper when not admin); ViewSuspended
  Services/IconCache.cs         System.Drawing Icon.ExtractIcon(path,0,32) → PNG → BitmapImage; stock app icon fallback
  Services/ProcessActions.cs    End (creation-time checked), open location, Properties (ShellExecuteEx "properties"), copy
  Services/AppSettings.cs, Format.cs
  ViewModels/Infrastructure.cs  ObservableObject, SlotCollection, ColumnLayout (shared widths, persisted with prefix), Heat
  ViewModels/ProcessTree.cs     pure grouping/sorting/search → ProcessRowData list (sections Apps/Background)
  ViewModels/ProcessSlot.cs, ProcessesViewModel.cs, NetworkViewModel.cs (NetworkSlot)
  Controls/TrendChart.cs        Polyline/Polygon area chart; Stroke/SecondStroke are DependencyProperties
  Views/ProcessesPage, PerformancePage, NetworkPage, SettingsPage
```

Data flow: `MonitorEngine` (thread pool, `Interval` default 1 s) → `SnapshotReady` → `MonitorHost` (keeps only the newest pending snapshot, `DispatcherQueue.TryEnqueue`) → `Updated` → the visible page calls its view model's `Refresh()`.

Processes page columns: Name | CPU | Memory (private WS) | Disk/I/O | Network (Mbit/s, admin) | Connections | Publisher | Path.
Network page columns: App/protocol | Local address | Remote host | State | Speed (down/up).

## 5. Verified facts (measured on Kaan's PC, 20 logical CPUs, ~300 processes)

- `SYSTEM_PROCESS_INFORMATION` x64 offsets in `NtProcessReader` match .NET `Process` values for 333/333 processes. One read: 3.7 ms (old per-process approach ≈ 65 ms, 42 ms of it `MainWindowTitle`).
- Task Manager's Memory column = **private working set**. "In use" = total − available − modified (= Task Manager "In Verwendung"). Example breakdown: in use 20.0 GB = apps 10.4 + kernel pools 3.4 + file cache 1.1 + drivers 0.06 + unattributed ~5.1 (shared images, page tables, driver-locked pages; RAMMap can split further).
- Full snapshot incl. network tables: 6–27 ms. `NetworkInterface.GetAllNetworkInterfaces()` is expensive → adapter list cached 10 s (took the app from ~19% to ~2.7% of one core).
- App cost at 1 s refresh: ~2.7% of one core visible (Processes page), ~3% minimized (no UI work while minimized); elevated with ETW ~1.1% on the Network page.
- ETW bandwidth: 25,000,000-byte download counted as 25,040,918 (TLS overhead). With curl capped at 16.8 Mbit/s, per-tick readings 16.3–16.8 (avg 16.6). **Without the `ControlTrace` flush before each sample, readings alternated 0 / double** (kernel delivers buffers ~1/s). Kernel TCP/IP events: `sport` = local port, `daddr`/`dport` = remote, for send and receive; `ProcessID` is correct.
- TraceEvent quirks: IPv6 UDP data class is spelled `UpdIpV6TraceData`; `TraceEventSession` has no Flush method (use `ControlTraceW(0, name, props, 3)`; properties = 120-byte `EVENT_TRACE_PROPERTIES` + name buffers, LogFileNameOffset @112, LoggerNameOffset @116). Kernel keywords work in a custom-named session on Win 8+. Sessions: `BetterTaskManager-Network`, `BetterTaskManager-Dns` (reused by name, so a crash doesn't leak them).
- DNS: `MSFT_DNSClientCache` (root\StandardCimv2) readable without admin, ~250 ms; `Entry` = name the app asked for. In a non-elevated test 14–18 of ~27 established remote TCP connections got a looked-up name. Browsers with DNS-over-HTTPS bypass the Windows cache → reverse DNS fallback. `Dns.GetHostEntryAsync(ip)` also forward-resolves and pollutes the cache → PTR-only `DnsQuery_W` is used.
- `IPAddress.ScopeId` throws for IPv4 — only read it for IPv6.
- Single-instance fix verified: with an elevated instance running, a standard-user launch exits cleanly (code 0). The WinForms preview crashed here because the elevated mutex/event denies medium-IL access.

## 6. Design decisions and UI rules

- **Rows are reusable slots** (`SlotCollection.Apply`): values are written into existing row objects; never clear/re-add. WinRT turns `ObservableCollection.Move` into remove+insert, which drops selection — so selection is tracked by **row key** and restored after each refresh (`RestoreSelection`).
- `ListView.ItemContainerTransitions` is emptied (rows update every second).
- Sections ("Apps (n)" / "Background processes (n)") and rows share one template with visibility toggles, because a template selector would not re-run when a slot changes kind.
- Grouping key: executable path (or image name); **each svchost stays separate** and is named "Service Host: <first service>"; only svchost gets that prefix.
- Heat tint: amber `#FFAA33` with alpha steps 0/26/46/70/96/124/156; thresholds per column in `ProcessSlot.Load`.
- End task = Task Manager semantics (no prompt for normal apps); verifies creation time before `Kill`; warns for `IsProcessCritical`; process-tree kill is a separate, confirmed action.
- Firewall: same rule names as WinForms; standard users get one UAC prompt per change via `--firewall-block/--firewall-unblock <path>` (helper exits before WinUI starts).
- Restart as administrator: new process gets `--wait-for-pid <old>`; the old one releases the mutex and closes.
- XAML gotchas: `{ThemeResource}` can only target DependencyProperties; `NavigationView.SettingsItem` is null until `Loaded`; WinUI TextBlock has no `FontFeatures`; keyboard accelerators on the root need `KeyboardAcceleratorPlacementMode.Hidden` or they show "Ctrl+1" tooltips; a lambda parameter named `_` is a real variable (`_ = new App()` inside `Application.Start(_ => …)` fails).
- Loopback traffic is excluded from bandwidth. UDP rows get the per-(pid, local port) sum, assigned to the first matching row only.
- Host names: a name the app looked up always beats reverse DNS; reverse names are labelled as such in the tooltip.

## 7. Status

Done and verified on screen: Processes (sections, groups, icons, heat, sort, search, context menu, selection persistence), Performance (CPU/Memory/Network charts, memory breakdown), Network (grouped connections, speed per app and per connection, totals, active-only filter), Settings (interval, theme, elevation, about), light/dark theme, single-instance handover to an elevated copy.

Built but not yet seen on screen or exercised by hand: Path column, Processes Network (Mbit/s) column in elevated mode, host names in the Fluent UI (verified in the Core harness only), End task, End process tree, firewall block/unblock on a real app, Restart as administrator from the UI.

Known limitations: English only; theme brushes resolved in code-behind (Performance legend) follow the app theme at page creation; per-app transfer totals reset when the app restarts (no persistence yet); IPv6 link-local addresses are not named by design.

## 8. Next: background history service (Portmaster-style)

Goal: keep recording connections and traffic while the window is closed, and show history in the UI. Not started.

1. New project `src/BetterTaskManager.Service` (console, `Microsoft.Extensions.Hosting.WindowsServices`), references Core, runs as LocalSystem so ETW works without UAC prompts.
2. Collector loop every 1–2 s: connections + `BandwidthMonitor` + `HostNameResolver` + process identity (path, description). Reuse `MonitorEngine` if practical; the service does not need windows/icons.
3. Storage: SQLite (`Microsoft.Data.Sqlite`) at `%ProgramData%\BetterTaskManager\history.db`, WAL mode. Tables: `connections` (first_seen, last_seen, app_path, app_name, pid, protocol, local/remote address+port, remote_host, bytes_in, bytes_out, last_state), `app_usage_daily` (date, app_path, bytes_in, bytes_out). Insert a connection on first sight, update it while it lives. Retention 30 days, pruned daily.
4. UI: History page reading the DB (read-only connection): per-app usage today / 7 days, searchable connection log with time filter. The Network page can show "total today" from the DB when the service runs.
5. Settings: "Record in the background" toggle → elevated helper `--install-service` / `--uninstall-service` (sc.exe create with the service exe path, start=auto; description; recovery). Show service status.
6. Privacy note in the UI: ProgramData is readable by local users; fine on Kaan's single-user PC, but say so.
7. Do not port the WinForms `NetworkHistoryStore` (CSV); it is superseded.

## 9. Later roadmap

1. German localization (resource-based; UI strings only — never translate process names, titles or paths; the WinForms app did and corrupted them).
2. Details view (all columns, user per process), startup apps, efficiency mode.
3. Installer/CI switch to the Fluent app (preview branch has Inno Setup + `windows-ci.yml` running `--self-test/--ui-smoke-test` on the WinForms exe), then retire WinForms.
4. Optional: ask-on-connect prompts need a WFP callout driver or block-by-default + blocked-event notifications (simplewall's approach); not planned yet.

## 10. Review findings on the WinForms builds (for reference)

- `main` (v1.0): Network grid header sort crashes on mixed number/text keys (de-DE parses `127.0.0.1` as 127001); history keeps only the latest snapshot (quoted timestamp never parses); per-app netsh calls on the UI thread freeze the window; History page unreachable.
- `preview-2`: fixed most of the above. Remaining issues: German translation layer rewrites user data (window titles, paths); Apps page flickers "Loading apps…" and loses scroll/sort every Live tick; single-instance crash after Restart as Admin (fixed in the Fluent app); tree-kill by default. "Memory" columns summed full working sets (double-counts shared pages) — the root of Kaan's "memory doesn't add up" complaint.
