# Nax-TaskManager (formerly Better Task Manager) — Fluent rebuild: handoff

Single source of truth for continuing this work in a new session. Read this before touching the code.

## 1. Goal and owner context

- Owner: Kaan (Naxterra). .NET developer; wants a partner, not a cheerleader; verify claims with live data.
- Goal: **replace Windows Task Manager** (numbers must reconcile with it) and get **Portmaster-style per-app network monitoring** without paying for Portmaster (Safing SPN not needed).
- UI must be **Fluent, not Excel-like, not laggy**. English UI for now; German localization later.
- GitHub (`github.com/Naxterra/Nax-TaskManager`, renamed from Better-Task-Manager on 2026-10-04; the old URL redirects) is only Kaan's **cloud backup**; nobody else uses it. Commit when asked; **do not push unless asked**.
- Kaan's sibling app **DiskLoom** (`D:\KI\Claude Code\DiskLoom`) is WinUI 3 / .NET 11 unpackaged; this rebuild copies its project setup and `Program.Main` pattern.

## 2. Repository state

- **Name (since 2.0.0-alpha.2, 2026-10-04): Nax-TaskManager**, exe `NaxTaskManager.exe`, service exe `NaxTaskManager.HistoryService.exe`, matching Kaan's other apps (Nax-Copy → NaxCopy.exe). Repo and local folder are `Nax-TaskManager`; project folders and namespaces are still `BetterTaskManager.*` (internal only).
  - Identifiers: mutex `Local\Naxterra.NaxTaskManager.SingleInstance`; settings `%LOCALAPPDATA%\NaxTaskManager\settings.json` (falls back to the old `BetterTaskManager\fluent-settings.json` once); crash log `crash.log` there; service `NaxTaskManagerHistory`, binaries in `%ProgramFiles%\Nax-TaskManager\HistoryService`, data `%ProgramData%\NaxTaskManager`; ETW sessions `NaxTaskManager-Network`/`-Dns`/`-History`.
  - Firewall rules are now named `Nax-TaskManager Block <hash>`; old `BetterTaskManager Block <hash>` rules (same hash) still count as blocked and are deleted on unblock.
  - Installer keeps the old AppId (so it upgrades the Better Task Manager install) but sets `UsePreviousAppDir=no`, installing to `%LOCALAPPDATA%\Programs\Nax-TaskManager`.

- Local clone: **`D:\KI\Claude Code\Nax-TaskManager`** (renamed from `Better-Task-Manager` on 2026-10-04) (the old `D:\KI\Codex\…` clone was deleted on Kaan's side).
- Branches: **`main` = this app** (fast-forwarded to `fluent-ui` on 2026-10-04; releases are cut from `main`); `codex/better-task-manager-preview-2` = the retired WinForms v1.1-preview. The WinForms project was removed from the tree in 2.0.0-alpha.4 (git history and release v1.1.0-preview.57 keep it).
- **Always push `fluent-ui` after committing.** On 2026-10-03 the only copy (the deleted Codex clone, never pushed) was gone; the branch was rebuilt by replaying this session's transcript (every Write/Edit and file-changing command, in order, on a fresh clone at `ff029c5`) and checked against every file read the transcript recorded (all matched). The replay tooling lives in the session scratchpad (`recover\replay.py`). Commit hashes changed in the process:
  - `e5d292d` WinUI 3 rebuild + Core collector (was 9a0a168)
  - `aa8e128` per-app network throughput (ETW) + Path column (was a1951e4)
  - `d7976b7` remote host names (was f3189f3)
  - `043c681` this handoff document (was f80acc2)
  - `7397db0` restart ETW traces stopped by another program; session-name prefix (was 3802e22)
  - `c93001e` background history service + History page (was 8c7fc1b)
  - then: release packaging for v2.0.0-alpha.1 (exe renamed to `BetterTaskManager.exe`, installer/scripts switched to the Fluent app)

## 3. Build, run, test

```powershell
dotnet build BetterTaskManager.slnx -c Release          # whole solution, currently 0 warnings
dotnet build src/BetterTaskManager.Fluent -c Debug
src\BetterTaskManager.Fluent\bin\Debug\net11.0-windows10.0.26100.0\win-x64\NaxTaskManager.exe --page Network
```

CI (`.github/workflows/windows-ci.yml`, every push incl. tags): build solution, classic WinForms self-test + UI smoke test, `publish-fluent.ps1`, 15-s History service console run (runner is elevated, so the kernel trace and SQLite run for real), `build-installer.ps1`, `test-installer.ps1` (separate test AppId: install, upgrade while the app runs (Restart Manager closes it), launch smoke test 10 s + crash-log check, uninstall), upload artifacts. **Any change to scripts/installer must keep CI in step**: v2.0.0-alpha.1 to alpha.3 all failed CI because the workflow still published the WinForms app (fixed 2026-10-04, first green run 37227895411).

Release (per-user Inno Setup, same AppId as the WinForms preview, so it replaces that install):
1. Bump `<Version>` in `src/BetterTaskManager.Fluent/BetterTaskManager.Fluent.csproj` and `src/BetterTaskManager.HistoryService/BetterTaskManager.HistoryService.csproj`; commit; **push `fluent-ui`**.
2. `scripts\publish-fluent.ps1` → `artifacts\BetterTaskManager-v<ver>-portable-win-x64` (+ .zip). Framework-dependent: needs the .NET 11 desktop runtime.
3. `scripts\build-installer.ps1` → `artifacts\BetterTaskManager-v<ver>-setup-win-x64.exe` + `SHA256SUMS-v<ver>.txt` (numeric file version `2.0.0.<prerelease number>`).
4. `git tag v<ver>` on `fluent-ui`, push the tag, `gh release create v<ver> --prerelease --target fluent-ui` with the zip, setup and checksums.
5. Install: `<setup>.exe /VERYSILENT /SUPPRESSMSGBOXES /NORESTART` (no UAC; installs to `%LOCALAPPDATA%\Programs\Better Task Manager`). Check the installed exe's ProductVersion.
- The installed history service is **not** updated by setup: after an upgrade, toggle "Record in the background" off/on (or click Turn on) to copy the new service build. A non-silent uninstall removes the service (one UAC prompt); silent uninstalls (what setup uses during upgrades) leave it.

- `--page Processes|Performance|Network|Settings` opens a page directly (used for screenshots).
- SDK: .NET 11 RC (`11.0.100-rc.1`), Windows App SDK 2.2.0, Windows SDK BuildTools 10.0.28000.2270.
- Settings: `%LOCALAPPDATA%\BetterTaskManager\fluent-settings.json` (saved on graceful close only). Crash log: `fluent-crash.log` in the same folder.

### Testing pitfalls (all hit in practice)

- **Single instance**: a running copy makes new launches hand over and exit. A running copy also locks `bin\Debug`; build with `-o <scratch folder>` instead of killing Kaan's window.
- **Elevated copies** cannot be closed, clicked or captured from a non-elevated shell or computer-use (UIPI). Ask Kaan to close them.
- **Exe name**: `NaxTaskManager.exe` for both dev and installed builds, sharing one single-instance mutex: while one runs, launching the other hands over to it. Launch dev builds by full path and check `Get-Process NaxTaskManager | select Path`; never close the installed copy without asking. computer-use grant: `naxtaskmanager.exe`.
- **UI checks without touching Kaan's input**: scratchpad `verify\uia.ps1` (UI Automation Invoke/Toggle by accessible name, e.g. "Sort by State") and `verify\rows.ps1` (row order). Header buttons carry `AutomationProperties.Name="Sort by …"`, rows announce their name. A test window that pops up may get clicked by Kaan; re-read state rather than trusting one read. The window may start hidden behind others: restore with `ShowWindow(h, 9)` + `SetForegroundWindow`.
- When the screen is busy (another session), capture the window with `PrintWindow(hwnd, dc, 2)` after `ShowWindow(h, 4)` (no focus steal).
- Admin-only features (ETW bandwidth, live DNS) were verified with a **separate elevated console harness** that references Core (`Start-Process -Verb RunAs`, writes results to a file); Kaan approves the UAC prompt. The harness lived in the session scratchpad; recreate it when needed (console exe, `ProjectReference` to Core, call `MonitorEngine.Start()`, set `Paused = true`, then drive `Collect()` once per second). **Always construct it as `new MonitorEngine("BTM-Test")`**: ETW session names are machine-wide, and a harness using the app's default names (`BetterTaskManager-Network`/`-Dns`) stops the running app's traces. That happened once and made Kaan's elevated window show "Admin" in the Network column. Since then the app restarts a lost trace within about 10 s and shows "Paused" with the reason in the tooltip while it waits. This was verified with an elevated harness that took over the sessions deliberately.
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
  Monitoring/MonitorEngine.cs   background loop → immutable MonitorSnapshot per interval; ctor takes the ETW session-name prefix
  Monitoring/Samples.cs         ProcessSample, ConnectionSample, FlowSample (per-socket bytes incl. UDP remote), MemoryBreakdown, SystemSample, MonitorSnapshot
  History/HistoryStore.cs       SQLite (Microsoft.Data.Sqlite): tables meta, connections, app_usage; write batch, prune, read queries
  History/HistoryRecorder.cs    snapshots → TCP rows (from the table) + UDP/TCP flows (from ETW) + bytes per (local day, app); flush every 10 s
  History/HistoryServiceControl.cs  service name/state, Install (copy to Program Files, sc create/config, failure actions, start), Uninstall

src/BetterTaskManager.HistoryService  (console exe, framework-dependent, ServiceBase — no Hosting package)
  Program.cs                    service mode, or `--console [--db path] [--seconds n]` for tests (own ETW prefix BetterTaskManager-HistoryTest)
  HistoryWindowsService.cs      OnStart: protected ACL on the data folder (SYSTEM/Admins full, Users read), log to service.log (1 MB, rotates)
  HistoryWorker.cs              MonitorEngine("BetterTaskManager-History", 2 s) → HistoryRecorder

src/BetterTaskManager.Fluent    (WinUI 3 unpackaged, self-contained WinAppSDK, DISABLE_XAML_GENERATED_MAIN)
  Program.cs                    custom Main: --firewall-block/--firewall-unblock helper mode, --wait-for-pid, single-instance mutex
  App.xaml(.cs)                 theme resources (heat/memory/chart brushes), styles; RestartElevated()
  MainWindow.xaml(.cs)          Mica, TitleBar with search box, NavigationView (LeftCompact), elevation InfoBar, Ctrl+1/2/3, Ctrl+F
  Services/MonitorHost.cs       owns MonitorEngine; coalesces snapshots to the UI thread; history buffers; search text;
                                firewall rule cache + SetBlockedAsync (elevated helper when not admin); ViewSuspended
  Services/IconCache.cs         System.Drawing Icon.ExtractIcon(path,0,32) → PNG → BitmapImage; stock app icon fallback
  Services/ProcessActions.cs    End (creation-time checked), open location, Properties (ShellExecuteEx "properties"), copy
  Services/AppSettings.cs, Format.cs
  Services/HistoryServiceSetup.cs  turn recording on/off; elevated helper `--install-history-service/--uninstall-history-service <resultFile>`
  ViewModels/Infrastructure.cs  ObservableObject, SlotCollection, ColumnLayout (shared widths, persisted with prefix), Heat
  ViewModels/ProcessTree.cs     pure grouping/sorting/search → ProcessRowData list (sections Apps/Background)
  ViewModels/ProcessSlot.cs, ProcessesViewModel.cs, NetworkViewModel.cs (NetworkSlot)
  Controls/TrendChart.cs        Polyline/Polygon area chart; Stroke/SecondStroke are DependencyProperties
  Views/ProcessesPage, PerformancePage, NetworkPage, HistoryPage, SettingsPage
  csproj target BundleHistoryService  builds the service and copies it to <app output>\HistoryService (also on publish)
```

Data flow: `MonitorEngine` (thread pool, `Interval` default 1 s) → `SnapshotReady` → `MonitorHost` (keeps only the newest pending snapshot, `DispatcherQueue.TryEnqueue`) → `Updated` → the visible page calls its view model's `Refresh()`.

Processes page columns (by app): Name | CPU | Memory (private WS) | Disk/I/O | Network (Mbit/s, admin) | Connections | GPU | Publisher | Path. All processes: Name | PID | Status | User | CPU | Memory | GPU | Description.
GPU = busiest engine per process from the PDH `\GPU Engine(*)\Utilization Percentage` counters (`Core/Native/GpuCounters`, no admin needed, about 1 to 9 ms per read; one wildcard query picks up new processes); header = busiest engine summed over all processes; groups sum their members (capped at 100).
Network page columns: App/protocol | Local address | Remote host | State | Speed (down/up).
Disk page (alpha.19): disks from PDH `PhysicalDisk(*)` (English names) + DriveInfo + IOCTL_STORAGE_QUERY_PROPERTY (model, bus, seek penalty = HDD/SSD), no admin. Per-process disk bytes from NtQuerySystemInformation class 148 (SystemFullProcessInformation; extension after 136-byte extended thread entries, verified 256 MB write = 256.1 MB); class 148 is denied without admin, so `NtProcessReader` falls back to class 5 and the History service (LocalSystem) adds `Disk` to the feed message. Image names in class 148 are NT paths, trimmed to the file name. Per-file list: `Core/Disk/DiskFileActivity`, kernel session `NaxTaskManager-Disk` (8 MB, DiskIO | DiskFileIO | Thread) only while the Files list is visible and elevated; names of files opened before the trace come from a short second session (`-Names`) whose stop makes Windows emit FileRundown; leftover session is stopped at elevated startup. Rates are 60 s averages like Resource Monitor.
Cache trim (alpha.23): `Core/Disk/CacheTrimmer` in the History service. Reads the NTFS change journal (FSCTL_READ_USN_JOURNAL, V0 input, ReturnOnlyOnClose, reasons DATA_OVERWRITE | DATA_EXTEND | CLOSE) of every fixed NTFS drive that already has a journal, every 5 s. Written files >= 256 MB are opened by file id (OpenFileById) with FILE_FLAG_NO_BUFFERING 30 s and 5 min after the close; opening alone makes Windows drop the file's cached pages (measured: 2 GB test file, Witcher 3 texture.cache 17.2 GB freed; works without admin for readable files, volume handles need admin). Files still open or mapped by a running program keep their pages. NTFS metadata paths (`\$`) are skipped. Console test: `--console --cache-trim`.

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
- Firewall (since alpha.4): **WFP filters, not Windows Firewall rules** (`Core/Firewall/WfpBlocker.cs`). On Kaan's PC Bitdefender Endpoint Security Tools is the registered firewall product, so netsh rules were created but not enforced (verified: blocked curl copy still connected). Persistent filters under provider `6f1b5e6a-…-4b11` / sublayer `…4b12` (weight 0x8000), one per IP version at `FWPM_LAYER_ALE_AUTH_CONNECT_V4/V6`, condition `ALE_APP_ID` = `FwpmGetAppIdFromFileName0(path)`, action BLOCK, filter key = SHA-256(lower path + layer) as GUID. Verified: blocked curl copy could not connect despite Bitdefender. Standard users cannot read WFP (access denied), so every change also writes `%ProgramData%\NaxTaskManager\blocked-apps.txt` (folder ACL: admins write, users read; `Core/DataFolder.cs`), which the unelevated app reads. Old `BetterTaskManager Block`/`Nax-TaskManager Block` netsh rules are still recognised and deleted on any change. Standard users get one UAC prompt per change via `--firewall-block/--firewall-unblock <path>`.
- Restart as administrator: new process gets `--wait-for-pid <old>`; the old one releases the mutex and closes.
- XAML gotchas: `{ThemeResource}` can only target DependencyProperties; `NavigationView.SettingsItem` is null until `Loaded`; WinUI TextBlock has no `FontFeatures`; keyboard accelerators on the root need `KeyboardAcceleratorPlacementMode.Hidden` or they show "Ctrl+1" tooltips; a lambda parameter named `_` is a real variable (`_ = new App()` inside `Application.Start(_ => …)` fails).
- Loopback traffic is excluded from bandwidth. UDP rows get the per-(pid, local port) sum, assigned to the first matching row only.
- Host names: a name the app looked up always beats reverse DNS; reverse names are labelled as such in the tooltip.

## 7. Status

2.0.0-alpha.2 (2026-10-04): Network page sortable (App, Remote host, State, Data, Speed; default App A→Z, persisted as `NetworkSortColumn/Descending`; connections inside a group keep a fixed order unless sorted by host or speed), new Data column (bytes since the app started watching), Pause button, idle speeds blank. Performance page "Free up memory" (trim app working sets; clear standby cache and empty all working sets need admin; result shows In use/standby/free before → after). `Core/Native/MemoryCleanup.cs` is the port of the WinForms actions; the non-elevated refusal path was verified, the actual cleanup actions were not run (they would have trimmed Kaan's running game).

Done and verified on screen: Processes (sections, groups, icons, heat, sort, search, context menu, selection persistence), Performance (CPU/Memory/Network charts, memory breakdown), Network (grouped connections, speed per app and per connection, totals, active-only filter), Settings (interval, theme, elevation, about), light/dark theme, single-instance handover to an elevated copy.

Built but not yet seen on screen or exercised by hand: Path column, Processes Network (Mbit/s) column in elevated mode, host names in the Fluent UI (verified in the Core harness only), End task, End process tree, firewall block/unblock on a real app, Restart as administrator from the UI.

Known limitations: English only; theme brushes resolved in code-behind (Performance legend) follow the app theme at page creation; per-app transfer totals reset when the app restarts (no persistence yet); IPv6 link-local addresses are not named by design.

## 8. Background history service (Portmaster-style) — verified 2026-10-04, installed on Kaan's PC

**Live feed (2.0.0-alpha.3)**: the service also publishes every sample on the named pipe `NaxTaskManager.NetworkFeed` (`Core/Feed/NetworkFeed.cs`): 4-byte length + source-generated JSON `NetworkFeedMessage` (per-process rates/totals keyed by PID + creation time, per-socket rates). DACL: SYSTEM/Admins full, Authenticated Users read only (so users cannot create rogue pipe instances once the service owns the name). The non-elevated app (`MonitorEngine.UseServiceFeed()`, called by MonitorHost when not elevated) only trusts the pipe if `GetNamedPipeServerProcessId` equals the service PID from `QueryServiceStatusEx`; it retries on any failure. While a reader is connected the service samples every 1 s (else 2 s). `SystemSample.PerProcessNetworkFromService` drives the UI wording. Data column totals from the service count since the service started watching each process.
Verified: non-elevated engine switched to the feed within 1 s; a 2 MiB/s-capped 25 MB curl download read avg ≈16.8 Mbit/s with its connection row; installed app (non-elevated) shows Data/Speed and "measured by the History service". The service recorded each 25,000,000-byte download as 25.04 MB (TLS overhead) with host speed.cloudflare.com; data folder ACL as designed. Per-tick feed rates jitter (1-s windows of two unsynchronised samplers); averages are right.
Note: VPN traffic is counted twice in "all apps" totals — once for the app and once for the WireGuard tunnel service that carries it (seen: IDM 4.0 GB and WireGuard 4.3 GB on the same day).
Pitfall: file-based `dotnet run x.cs` harnesses default to AOT settings (reflection-based JSON and COM/WMI disabled); add `#:property PublishAot=false` to mirror the app.


Implemented as planned below, with these decisions:
- **Journal mode DELETE, not WAL**: WAL readers need write access to `-shm`; standard users only get read access to the data folder.
- **Data folder ACL** is set by the service (protected: SYSTEM/Admins full, Users read), because ProgramData lets any user create files in subfolders.
- **Binaries run from `%ProgramFiles%\Nax-TaskManager\HistoryService`** (install copies them there): a LocalSystem service must not run from a user-writable folder like the dev `bin`. Updating = toggle on again (stops, copies, reconfigures, starts).
- Service name `BetterTaskManagerHistory`, start delayed-auto, restart on failure (60 s ×3). Uninstall keeps `history.db`.
- **App key** = lowercase path with version numbers replaced by `*` (`app-*`, `claude_*_x64__…`), so updates don't split an app's history; svchost = `svchost:<first service>`; unknown/exited = `pid:<n>`.
- Loopback excluded; TCP rows come from the connection table (idle connections count), UDP rows only from ETW flows (only they know the remote side, e.g. QUIC), UDP flow ends after 60 s idle.
- UI: History page (Ctrl+4) = apps by data used (range Today / 7 / 30 days) + connection log (latest 1000, search box filters via SQL LIKE, auto-refresh 10 s); Settings → History card with toggle + privacy note.

Verified (2026-09-27, non-elevated console run): 50 TCP connections across 14 apps recorded and read back. Elevated/LocalSystem verification: see the live-feed paragraph above.

Original plan:

1. New project `src/BetterTaskManager.Service` (console, `Microsoft.Extensions.Hosting.WindowsServices`), references Core, runs as LocalSystem so ETW works without UAC prompts.
2. Collector loop every 1–2 s: connections + `BandwidthMonitor` + `HostNameResolver` + process identity (path, description). Reuse `MonitorEngine` if practical; the service does not need windows/icons.
3. Storage: SQLite (`Microsoft.Data.Sqlite`) at `%ProgramData%\BetterTaskManager\history.db`, WAL mode. Tables: `connections` (first_seen, last_seen, app_path, app_name, pid, protocol, local/remote address+port, remote_host, bytes_in, bytes_out, last_state), `app_usage_daily` (date, app_path, bytes_in, bytes_out). Insert a connection on first sight, update it while it lives. Retention 30 days, pruned daily.
4. UI: History page reading the DB (read-only connection): per-app usage today / 7 days, searchable connection log with time filter. The Network page can show "total today" from the DB when the service runs.
5. Settings: "Record in the background" toggle → elevated helper `--install-service` / `--uninstall-service` (sc.exe create with the service exe path, start=auto; description; recovery). Show service status.
6. Privacy note in the UI: ProgramData is readable by local users; fine on Kaan's single-user PC, but say so.
7. Do not port the WinForms `NetworkHistoryStore` (CSV); it is superseded.

## 8b. Network phase (2.0.0-alpha.5), modelled on Portmaster

Portmaster findings (source read 2026-10-04: service/network/connection.go, netquery/database.go, profile/special.go, profile/fingerprint.go, process/tags/*, resolver/ipinfo.go, netutils/ip.go): apps are profiles matched by fingerprints (tag > cmdline > env > path), svchost per service (suffix `_xxxx` stripped), Store apps by package name + publisher; special profiles "Operating System", "Other Connections", "Network Noise", "System DNS Client"; IP→domain is cached **per profile** from the app's own DNS answers; DNS requests are connection rows of type "dns"; IP scopes HostLocal/LinkLocal/SiteLocal (incl. CGNAT)/Global/Local+GlobalMulticast/Invalid plus inbound/outbound; history is SQLite with per-profile retention; bandwidth comes from its kernel driver (paid tier for history/bandwidth). No special VPN handling found.

What we adopted without a driver:
- `Core/Monitoring/AppIdentityRules.cs` (keys/names; history rekey = schema 2).
- DNS-Client ETW events **3006/3008 carry the requesting app's PID** (verified with curl: 3008 pid = curl, internal 3009–3020 pid = Dnscache). `HostNameResolver.Resolve(address, pid)` prefers the app's own answer; `QueryAnswered` feeds History DNS rows (protocol "DNS", remote_address = name, local_address = answers; excluded from connection counts).
- `Core/Network/IpScopes.cs` + TCP direction (local port is a listening port of the same PID) → Network "Scope" column, history `scope`/`inbound` (schema 3).
- Kernel trace also enables the Process keyword (`ProcessStart/DCStart/Stop`, `KernelImageFileName` → `DevicePaths.ToDosPath`), kept 10 min after exit → History names short-lived processes.
- `Core/Network/VpnTunnels.cs`: session-0 processes named after tunnel tech (wireguard, openvpn, wintun, tailscale, warp-svc…) are tunnels → badge + excluded from all-apps totals (history `app_usage.tunnel`, schema 4). Verified on Kaan's PC: only "Windscribe Wireguard Tunnel" matches (not the Windscribe UI or "Windscribe Service").
- Feed smoothing: 3-s average of service messages (measured ±5 % instead of ±40 %).
- Service update prompt: `HistoryServiceControl.NeedsUpdate` compares ProductVersion of bundled vs installed service.

## 8c. Session status 2026-10-04 (late) — read this first when continuing

Kaan asked to "do all" of: housekeeping, testing pass, the open gaps, network work modelled on Portmaster, German localization, Details view / startup apps / efficiency mode.

**Done, released and installed (2.0.0-alpha.4 → alpha.6; History service runs alpha.6; CI green on `main`):**
- `main` carries the app (fluent-ui merged and deleted); WinForms project removed; README/SECURITY/CHANGELOG rewritten.
- UI testing pass via UI Automation (no mouse input): End task, End process tree (now also in the command-bar overflow, context menus open via keyboard), Block/Allow, all three memory cleanup actions (elevated), Restart as administrator. Found and fixed: History app rows lost in the branch recovery (PowerShell here-string CRLF mismatch in replay call #232, the only such edit), UDP rows per local port, "0 KB", log order.
- **WFP blocking** (Bitdefender made netsh rules ineffective) — see section 6.
- Network phase (section 8b): app identity rules, short-lived process names, scope/direction, per-app DNS attribution + DNS rows (hidden behind the History "DNS lookups" toggle), VPN tunnel marking, feed smoothing, service update prompt.

**German localization — done (2.0.0-alpha.7):**
- `Strings/<lang>/Resources.resw` (en-US, de-DE) are the source of truth and compile into `Nax-TaskManager.pri` (`NaxTaskManager.pri` before alpha.15). XAML uses `x:Uid="<Page>_<n>"` (keys `<Uid>.<Property>`; attached properties as `<Uid>.[using:Microsoft.UI.Xaml.Controls]ToolTipService.ToolTip` / `<Uid>.[using:Microsoft.UI.Xaml.Automation]AutomationProperties.Name`). Code uses `Loc.Get(key)` / `Loc.F(key, args)` (`Services/Loc.cs`, MRT Core `ResourceLoader`). Product name, "CPU", "TCP", "UDP" are not translated.
- Core keeps English wording (it also writes the history DB); `Loc.State`, `Loc.Scope`, `Loc.AppName` ("Operating System", "Service Host: ", "Exited process (PID "), `Loc.NetworkStatus` and `Loc.CleanupFailure` (uses `CleanupResult.Failure`/`Code`) map it for display. A new Core string shown in the UI needs a mapping there.
- `Loc.ApplyLanguage(setting)` runs in `Program.Main` before XAML (`PrimaryLanguageOverride` + UI culture); `--lang de-DE|en-US` overrides for tests. Settings → Language ("System"/"en-US"/"de-DE") applies after "Restart now".
- Adding a string: add the key to **both** resw files. Wording follows the German Task Manager (Prozessstruktur beenden, In Verwendung, Zugesichert, Datenträger, Herausgeber, Dateipfad öffnen).
- Verified with `--lang` runs and PrintWindow captures of every page in both languages, plus menu/flyout names via UIA (flyouts are separate popups that PrintWindow does not capture). German forced wider defaults: Processes Memory 130 / Disk 140, History protocol column 260.
- Kaan's PC runs German Windows, so "System" shows German.

**Efficiency mode, Details, Startup apps — done (2.0.0-alpha.8):**
- Core `Native/ProcessControl` reads priority + efficiency (one limited handle; `GetProcessInformation(ProcessPowerThrottling)`) and changes them after a creation-time check. Efficiency on = EcoQoS (control+state = execution speed) + idle priority; off = explicit no-EcoQoS + normal priority, like Task Manager. `Native/ProcessUsers` reads owners via `WTSEnumerateProcessesEx` (standard users only get their own processes' owners). `NtProcessReader` marks a process suspended when all threads wait with reason Suspended (thread array at entry+256, 80 bytes each, state @68, wait reason @72). `MonitorEngine.ReadProcessDetails` (off in the service): priority/efficiency every 2 s, owners every 5 s, ~1 ms per refresh; `InvalidateProcessDetails()` after a change.
- Processes: leaf (efficiency, all members) / pause (suspended) icons, Efficiency mode toggle in overflow + context menu (disabled for critical processes).
- Details page (Ctrl+5): flat list, sortable/resizable/searchable, Set priority (radio menu + confirmation), efficiency, end task (this PID only) / tree.
- Startup apps page (Ctrl+6), Core `Startup/StartupApps`: HKCU Run, HKLM Run (64 and 32-bit), user and common Startup folders (.lnk via IShellLinkW; advertised shortcuts have no target and fall back to the .lnk). Enabled flag = `Explorer\StartupApproved\{Run,Run32,StartupFolder}`; first byte odd (03) = disabled with FILETIME at bytes 4–11, 02 = enabled, missing = enabled. All-users entries change through the elevated helper `--startup-enable|--startup-disable <Source> <Name>`. Not covered: startup impact.
- alpha.9: packaged/Store startup tasks (`Startup/PackageStartupTasks`): `PackageManager.FindPackagesForUser("")`, manifests with `windows.startupTask` (Executable on the extension or its Application; duplicate TaskIds listed once), names via `SHLoadIndirectString` for `ms-resource:`. State under HKCU `...\AppModel\SystemAppData\<family>\<TaskId>`: State 0/1 disabled, 2 enabled, 3/4 policy. **Observed, undocumented:** `ShowNotification=1` with State 2 did not start at sign-in and Task Manager shows it disabled (Apple Mobile Device), so it counts as off. Enable writes State=2, UserEnabledStartupOnce=1, ShowNotification=0; disable writes State=1 + LastDisabledTime. Shortcuts with only an environment-variable target resolve via `GetPath(SLGP_RAWPATH)`. BIOS time = `HKLM\...\Session Manager\Power\FwPOSTTime` (ms). Test `ui-startup-package.ps1` exports and restores the Terminal key.
- `--test-instance` skips the single-instance mutex, so UI tests run beside Kaan's open window; `snap.cs` takes a PID.
- alpha.14: **Trace buffers and boot memory report.** TraceEvent's default `BufferSizeMB` is 64 (per its XML docs) and kernel session buffers are non-pageable, so each session cost up to 64 MB. Now 16 MB (kernel network) and 4 MB (DNS); an elevated app with the History service running calls `MonitorEngine.PreferServiceFeed()` (no own traces, falls back after 20 s without feed). Measured on Kaan's PC: pool tag `EtwB` 367 MB to 131 MB; a 4 x 40 MB unthrottled download at 226 Mbit/s was recorded as 160,234,304 bytes (100.1%), so no events lost. `Core/Native/PoolTags` (NtQuerySystemInformation class 22, works without elevation; `FindDrivers` scans System32\drivers for the tag bytes). `HistoryService/BootMemoryReport` writes `%ProgramData%\NaxTaskManager\boot-memory.txt` at uptime 2/5/10 min, only when the service starts within 10 min of boot; console test: `--console --db <tmp> --seconds 25 --boot-report <file>`. Other kernel memory owners seen: PktMon.sys 156 MB (not a capture; `pktmon status` says not running), Bitdefender (bddci4/Ignisv2/atc.sys) about 240 MB, fltMgr name cache about 120 MB. Do not try to enumerate ETW sessions with an elevated script: the safety classifier stopped that twice.
- alpha.13: **Processes and Details merged.** One `ProcessesPage` hosts both layouts, switched by a `SelectorBar` ("Nach App" grouped / "Alle Prozesse" flat); the mode persists in `AppSettings.ProcessFlatView`. The flat layout keeps `DetailsViewModel`/`DetailSlot` (now the flat VM) but **priority was removed** (column, sort, Set priority action, `PriorityName`, `PriorityRank`) per Kaan. Command bar and context menu act on `CurrentTarget()` (grouped: the selected group's processes; flat: the one process). `DetailsPage.*` deleted, Details nav item removed, Startup is now Ctrl+5. `Dialogs.cs` is left unused. **Search fix (alpha.12):** `DetailsViewModel.Matches` no longer matches the owner (Kaan's name "Naxterra" contains "nax", which flooded results); grouped `ProcessTree` shows "PID <n>" on single-process rows.
- alpha.10: **Replace Windows Task Manager** (`Native/TaskManagerReplacement`). Same mechanism as Sysinternals Process Explorer: HKLM `SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options\taskmgr.exe` `Debugger` = quoted app path. Windows then launches the app for every taskmgr.exe start and passes the original taskmgr command line as an argument, which Main ignores (no helper flag matches). Settings toggle (System integration), UAC-gated; `OtherTarget` shows and preserves another tool's entry; `DisableIfOurs` on uninstall. Helper modes `--replace-taskmanager-on <exe>` / `--replace-taskmanager-off [<exe>]`. Installer `[UninstallRun]` restores via `ShouldRestoreTaskManager` (skips silent upgrades so the choice survives). The replacement launches without taskmgr's auto-elevation, so it opens in limited mode. NOTE: the safety classifier repeatedly blocked authoring/testing this (IFEO Debugger is a dual-use technique); a false-positive appeal was filed. The live redirect was not scripted end to end; verify by toggling in Settings and launching Task Manager normally.
- alpha.15 (2026-10-05): exes renamed to `Nax-TaskManager.exe` and `Nax-TaskManager.HistoryService.exe` (AssemblyName; Kaan's rule: names and exes are Nax-Xxx with a hyphen). Namespaces, project names, `BetterTaskManager.Core`, service key `NaxTaskManagerHistory`, data folders `NaxTaskManager`, ETW session prefix and release asset names are unchanged. Setup: `CloseApplicationsFilter` covers both exe names; `[Run]` entry `ShouldMoveTaskManagerReplacement` re-points the IFEO `Debugger` from `{app}\NaxTaskManager.exe` to the new exe via the elevated `--replace-taskmanager-on` helper (Kaan had the replacement on). `HistoryServiceControl.Install` deletes `NaxTaskManager.HistoryService.*` / `BetterTaskManager.HistoryService.*` leftovers after copying; `sc config` already switches `binPath`.

**Out of scope, to be explained to Kaan:** ask-before-connect prompts need a WFP callout driver (kernel driver signing) — Portmaster has one, we deliberately do not.

**Test tooling (scratchpad `verify\`, recreate if the session changed):** `uitest.psm1` (Start-TestApp with `--test-instance --lang --page`, Find-El/Invoke-El/Select-Nav/Set-Search/Select-Row/Save-Snap by PID), `ui-efficiency.ps1`, `ui-details.ps1`, `ui-startup.ps1` (throwaway HKCU Run entry, removed afterwards), `ecostate.cs`, `uia.ps1` (Invoke/Toggle by accessible name), `rows.ps1` (row order), `snap.cs` (PrintWindow capture; run with `dotnet run --file`, not from a project folder), `ui-endtask.ps1` (End task/tree on renamed ping/cmd copies), `ui-admin.ps1` (elevated: WFP block/allow on a curl copy + cleanup; closes/reopens Kaan's app only with his consent), `install-release.ps1` (elevated install + service update), `feedclient*.cs`, `e2e.cs` (reads the service DB). File-based harnesses need `#:property PublishAot=false`.

## 9. Later roadmap

1. ~~German localization~~ — done in 2.0.0-alpha.7 (see 8c).
2. Details view (all columns, user per process), startup apps, efficiency mode.
3. ~~Installer/CI switch to the Fluent app, then retire WinForms~~ — done in 2.0.0-alpha.1 to alpha.4.
4. Optional: ask-on-connect prompts need a WFP callout driver or block-by-default + blocked-event notifications (simplewall's approach); not planned yet.

## 10. Review findings on the WinForms builds (for reference)

- `main` (v1.0): Network grid header sort crashes on mixed number/text keys (de-DE parses `127.0.0.1` as 127001); history keeps only the latest snapshot (quoted timestamp never parses); per-app netsh calls on the UI thread freeze the window; History page unreachable.
- `preview-2`: fixed most of the above. Remaining issues: German translation layer rewrites user data (window titles, paths); Apps page flickers "Loading apps…" and loses scroll/sort every Live tick; single-instance crash after Restart as Admin (fixed in the Fluent app); tree-kill by default. "Memory" columns summed full working sets (double-counts shared pages) — the root of Kaan's "memory doesn't add up" complaint.
