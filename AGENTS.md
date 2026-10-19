# Repository Guidelines

## Project Overview
RO3Emu is a GPL-3.0 server emulator for the **Ragnarok Online 3 PC client** (C#, .NET 10, SQLite). Goal: a faithful 1:1 reproduction of the original server first; rule changes later. Anything the client cannot prove is marked *(inferred)* in code and docs. Game data is exported from the user's own client into the gitignored `data/` folder; the client, its DLLs, `data/`, `re/` and logs are never committed.

## Architecture & Data Flow
- **Hosting:** `src/Ro3Emu.Server/ServerHost.cs` builds an ASP.NET Core `WebApplication`; ~150 explicit `AddSingleton` registrations. Hosted services start in order `GameDataPreload → GameLoopTickers → GameLoop → MicroGateServer/GameGateServer → GameLoopQuiesceService` (+ `DatabaseBackup`, optional `VoiceGateServer`). Minimal-API routes in `SdkEndpoints.Map`, `MetricsEndpoints.Map`, `/ro3emu/*` (admin routes loopback-only).
- **Single game-loop thread:** `Game/GameLoop.cs` runs a 50 ms tick with a custom `SynchronizationContext` and `IGameTicker`s. **All game state is owned by this thread; services take no locks.** Work enters via `loop.InvokeAsync(...)`. Admission/reservations live in `Game/Ownership/` (`OwnerRegistry`, `ReservationKey`).
- **Networking:** `Net/GateListener` (abstract `BackgroundService`) accepts TCP and runs one `Net/PhonestChannel` per client. Frame = `int32 LE len | uint16 LE rpc index | protobuf`, RC4 via `StreamConvertor` after the `GateCrypto` handshake; sends go to an unbounded non-blocking queue.
- **Game gate message path:** `GateListener → PhonestChannel → GameGateServer.OnFrameAsync` (rpc 1/2 handshake, 3 `ConnectServer` → `AccountStore.ResolveEngineSign` → `SessionRegistry.SignInAsync`, 4 `EntityMessageIn` → `loop.InvokeAsync`) → `GameMessageHandler.HandleAsync` → `MessageRoutes.Admit` (Economic/Runtime/Account/Role/Realm reservation) → interceptors → handler lambda from an `IMessageModule.Map` (`Game/Messages/*Messages.cs`) → `XxxReq.Parser.ParseFrom(body.Span)` → service → `session.SendGameAsync(msgId, IMessage)`. Unrouted ids get `MsgCatalog.EmptyResponse`.
- **Micro gate:** rpc 33 `MsRequest` → `MicroServiceRouter.InvokeAsync(service, method, msgpack)` → `IMicroService` in `Micro/` (MessagePack used untyped: `Dictionary<string, object?>`).
- **Persistence:** one SQLite file `<DataDir>/ro3emu.db`, WAL + `synchronous=FULL` (`Accounts/AccountStore.cs`); all writes go through a single DB writer thread. Schema is created only by `Accounts/Persistence/DatabaseSchema.Plans()` — stores never create tables. `RealmLock` = one server per data dir. Backups in `<data>/backups`.
- **Game data:** `Data/GameData.cs` reads JSON tables from `<GameDataDir|DataDir>/config/*.json` produced by `Ro3Emu.DataExport`.
- **Codegen:** `<Protobuf>` items in `Ro3Emu.Server.csproj` compile `Protos/phonest/*.proto` (ns `Phonest.Proto`) and `Protos/romsg/MG_*.proto` (~140 files, `GrpcServices="None"`); generated C# lives only in `obj/`. Protos are regenerated from the client's `LocalProto/proto.pb.bytes` with `tools/ProtoDump`.

## Key Directories
| Path | Purpose |
|---|---|
| `src/Ro3Emu.Server/` | The emulator (Web SDK). `Game/` (~300 flat `*Service.cs`, partials `Class.Aspect.cs`; subfolders `Messages/`, `Messaging/`, `Ownership/`, `Persistence/`), `Net/`, `Micro/`, `Maps/` (`World`, `NavGrid`, AOI), `Accounts/`, `Economy/`, `Data/`, `Http/`, `Voice/`, `Diagnostics/`, `Protos/` |
| `src/Ro3Emu.Bot/`, `BotCli/`, `LoadTest/` | `BotClient` speaks the real protocol; stdin-driven bot; load scenarios S1–S5 |
| `src/Ro3Emu.Patcher/`, `Launcher/` | Chunked updater (`Updater`, `FastCdc`, `Manifest`); WPF+WebView2 player launcher (`net10.0-windows`) |
| `src/Ro3Emu.ClientHooks/`, `Loc/` | `netstandard2.1`, **Nullable disabled**, built against the client's `ro3_Data\Managed` DLLs (`$(ClientManaged)` / `RO3_CLIENT_DIR`). Injected into the Unity client via Mono.Cecil |
| `src/Ro3Emu.ClientTools/`, `Setup/`, `DataExport/` | Client patcher CLI; one-click `RO3Emu.exe` (export → run → patch → launch); client-table exporter (KeraLua) |
| `tests/Ro3Emu.Tests/` | ~200 xUnit files for Server/Bot/LoadTest; `Fixtures/GameData/` synthetic tables; `Snapshots/` golden files |
| `tests/Ro3Emu.Setup.Tests/`, `Ro3Emu.Patcher.Tests/` | Installer/export tests; patcher tests |
| `tools/` | `LuaDecode/`, `ConfigExport/`, `ProtoDump/`, `unluac-batch/`, `WorldData/`, `Loc/`, `ClientTextures/`, `EconomyProbe/`, `release/` (`pack.ps1`, `smoke.ps1`), `run-server.ps1`, `patch-client.ps1`, `test-summary.ps1` |
| `docs/` | `guide/` (running), `systems/` (29 per-system notes), `research/` (client evidence), `site/` (GitHub Pages) |
| `prebuilt/client/` | Committed `Ro3Emu.ClientHooks.dll`, `Ro3Emu.Loc.dll`, `source.sha256` used by CI/release when no client is present |

## Development Commands
```powershell
# Client-free checks (what CI runs; always run these before claiming done)
dotnet build src/Ro3Emu.Server/Ro3Emu.Server.csproj -c Release
dotnet build tools/LuaDecode/LuaDecode.csproj -c Release
dotnet build tools/ConfigExport/ConfigExport.csproj -c Release
dotnet test tests/Ro3Emu.Tests -c Release --filter "Category!=Slow"
dotnet test tests/Ro3Emu.Setup.Tests -c Release --filter "Category!=Slow"
dotnet test tests/Ro3Emu.Patcher.Tests -c Release

# One system only (docs/systems/*.md lists a filter per system)
dotnet test tests/Ro3Emu.Tests -c Release --filter "FullyQualifiedName~GuildTests"
# Serial run for flaky-port investigations
dotnet test tests/Ro3Emu.Tests -c Release --settings tests/Ro3Emu.Tests/serial.runsettings

# Run by hand (needs exported data)
dotnet run --project src/Ro3Emu.Setup -c Release -- export C:\Games\ROO_PC data
dotnet run --project src/Ro3Emu.Server -c Release
dotnet run --project src/Ro3Emu.ClientTools -- play C:\Games\ROO_PC http://127.0.0.1:5380

# Wrappers
start.cmd / start.ps1 [-ClientDir X] [-Reexport] [-Join [-Server addr]]   # Windows one-click
./server.sh [--client DIR] [--host ADDR] [--gm ACCT] [--service] [--reexport]  # Linux
tools/run-server.ps1            # rebuild + restart loop, logs to data/logs/
tools/patch-client.ps1          # rebuild hooks/loc and patch the client
pwsh tools/release/pack.ps1 -UsePrebuiltClientDlls -Version vX.Y.Z   # then tools/release/smoke.ps1
dotnet run -c Release --project src/Ro3Emu.LoadTest -- --scenario S2 --server-local --bots 300 --duration-seconds 600 --output .loadtest/s2
```
- Full-solution build needs the client DLLs: pass `-p:ClientManaged=<client>\ro3_Data\Managed` or set `RO3_CLIENT_DIR`. Without a client, build only `Ro3Emu.Server` and the tools above.
- No linter or `dotnet format` runs in CI; `.editorconfig` is the only style authority.

## Code Conventions & Common Patterns
- **Formatting (`.editorconfig`):** UTF-8, spaces, 4-space indent, C# max line 120; 2-space for json/csproj/slnx/yml/js/html/css. `*.sh` LF, `*.cmd` CRLF.
- **Nullable + ImplicitUsings on** everywhere except `ClientHooks`/`Loc` (`netstandard2.1`, Nullable off, unsafe). No `TreatWarningsAsErrors`, no analyzer packages.
- **DI:** constructor injection with primary constructors — `public sealed class FooService(Dep a, ILogger<FooService> log)`. Classes are `sealed` by default. Optional deps are nullable with `= null` default so tests can `new` them directly. Circular deps are wired after `Build()` (e.g. `MountService.Changed`).
- **Threading rule (non-negotiable):** any game-state mutation runs on the game loop (`loop.InvokeAsync`). Never add locks to game services; never touch state from a hosted service or HTTP handler directly.
- **Async:** `Task`/`ValueTask`; continuations stay on the loop; per-connection reader awaits each message to preserve order.
- **Errors:** narrow filters (`catch (Exception e) when (e is IOException or SocketException ...)`); fail fast at startup with `InvalidOperationException` (duplicate routes, overlapping ranges).
- **Logging:** `ILogger<T>`, structured templates (`"GameGate: {Remote} ..."`), console only; `MetricsLoggerProvider` and `RecentLogBuffer` are extra providers.
- **Naming:** message ids are `public const uint` on the owning service (`AchievementService.Sync = 5019`); shared ids in `Game/MsgId.cs`. Large classes split into partials `Class.Aspect.cs` (`WorldCoreService.Movement.cs`). Handlers parse with `XxxReq.Parser.ParseFrom(body.Span)` and reply with `session.SendGameAsync(msgId, IMessage)`.
- **Evidence-first:** doc comments cite client symbols, Lua paths (`re/clientB/lua-src/...lua:N`), proto messages (`src/Ro3Emu.Server/Protos/romsg/MG_*.proto`) or audit tags. Unprovable rules get *(inferred)*. Follow `CONTRIBUTING.md` and the formats in `docs/research/`, `docs/systems/`.
- **Persistence:** new tables go in `DatabaseSchema.Plans()`; stores must not `CREATE TABLE`. Durable payouts/obligations follow `docs/systems/durable-settlement.md`.
- **Config:** `appsettings.json` section `"Ro3Emu"` → `ServerOptions` (`PublicHost` 127.0.0.1, `HttpPort` 5380, `MicroGatePort` 20000, `GameGatePort` 33100, `DataDir` `data`, `GameDataDir`, `PreloadGameData`, `TrafficLog`, `WalkCheck`, `GmAccounts[]`...). Override via `--Ro3Emu:Key=value` or `Ro3Emu__Key` env vars. Tests override through `ServerHost.Create(args, Action<ServerOptions>)`.

## Important Files
- Entry/host: `src/Ro3Emu.Server/Program.cs`, `ServerHost.cs` (DI + endpoints + `GameLoopTickers`), `ServerOptions.cs`, `appsettings.json`
- Network: `Net/PhonestChannel.cs`, `Net/GateListener.cs`, `Game/GameGateServer.cs`, `Micro/MicroGateServer.cs`
- Dispatch: `Game/GameLoop.cs`, `Game/GameMessageHandler.cs`, `Game/Messaging/MessageRoutes.cs`, `Game/Messages/*Messages.cs`, `Game/MsgCatalog.cs`, `Game/MsgId.cs`
- Storage/data: `Accounts/AccountStore.cs`, `Accounts/Persistence/DatabaseSchema.cs`, `Data/GameData.cs`
- Other entry points: `src/Ro3Emu.Setup/Program.cs`, `Ro3Emu.ClientTools/Program.cs`, `Ro3Emu.BotCli/Program.cs`, `Ro3Emu.LoadTest/Program.cs`, `Ro3Emu.Launcher/App.xaml.cs`
- Build/CI: `global.json`, `Ro3Emu.slnx`, `.editorconfig`, `.github/workflows/{ci,tests,release,pages}.yml`, `.github/pull_request_template.md`, `CONTRIBUTING.md`
- Read before touching systems code: `CONTRIBUTING.md`, `docs/guide/testing.md`, `docs/README.md`, `docs/systems/durable-settlement.md`, then `docs/systems/<area>.md` + matching `docs/research/*-evidence.md`

## Runtime/Tooling Preferences
- **.NET SDK 10.0.300** (`global.json`, `rollForward: latestFeature`); `net10.0` everywhere, `net10.0-windows` for Launcher, `netstandard2.1` for ClientHooks/Loc. No `Directory.Build.props` — settings live per `.csproj`.
- `Ro3Emu.Server.csproj` deliberately sets `ServerGarbageCollection=false`, `ConcurrentGarbageCollection=true`, `AllowUnsafeBlocks` — keep them.
- Node is only used for `tools/check-site.mjs`, `tools/WorldData/*.js`, `tools/Loc/*.js`; Java for `tools/unluac-batch` (bring your own `unluac.jar`). PowerShell 7 (`pwsh`) for `tools/*.ps1`.
- Env vars: `RO3_CLIENT_DIR` (client folder), `RO3EMU_HOME` (Setup home), diagnostics `RO3EMU_LOOPWAIT_DUMP`, `RO3EMU_SNAPSHOT_VERIFY=1`, `RO3EMU_MSG_COVERAGE`, `RO3EMU_SQL_CALLSITES`, `RO3EMU_OWNER_AUDIT`, `RO3EMU_DUMPTEX`.
- CI: `tests.yml` (ubuntu, every push/PR, SDK from `global.json`), `ci.yml` (windows, builds Server + tools, Patcher tests), `release.yml` (tag `v*`, `pack.ps1 -UsePrebuiltClientDlls`), `pages.yml` (`docs/site/**`, `node tools/check-site.mjs`). Refresh `prebuilt/client` with `pack.ps1 -UpdatePrebuiltClientDlls` whenever ClientHooks or Loc change.

## Testing & QA
- **Framework:** xUnit 2.9.3 + `Microsoft.NET.Test.Sdk` + `coverlet.collector` (installed, no threshold enforced). No Moq/FluentAssertions — plain `Assert.*` and hand-written fakes (`FileRepoHandler : HttpMessageHandler`, `ManualClock : TimeProvider`). `<Using Include="Xunit" />` is global.
- **Naming:** `public sealed class FooTests`, methods are PascalCase sentences, e.g. `ANewRoleEntersTheFixturesTownWithItsStarterKit()` — not `Method_Scenario_Expected`. Partials per topic: `SkillCombatTests.BossAttacks.cs`. `IAsyncLifetime` for async setup.
- **Helpers (`tests/Ro3Emu.Tests/`):** `SyntheticHost.StartAsync(o => ...)`, `SyntheticData.Dir`/`.Town`, `ClientData` + `[ClientDataFact]`/`[ClientDataTheory]`, `TestPorts.FreePort()` (avoids 5380/20000/33100/33101), `FixtureSql`, `WorldSettle`, `HostZoneOverride`.
- **Rules:**
  - Prefer the synthetic fixture (`tests/Ro3Emu.Tests/Fixtures/GameData`, runs in CI). Use `[ClientDataFact]` only when asserting on real exported content (skipped in CI; simulate with `RO3EMU_NO_CLIENT_DATA=1`).
  - **Always set `o.GameDataDir = SyntheticData.Dir`**, even for a bare `AccountStore`; otherwise local `data/` leaks in and CI fails.
  - Fixture values are invented — never copy client rows; add only the fields the code reads. `SyntheticFixtureTests` preloads every table and catches malformed fixtures first.
  - Whole-server tests: free loopback ports, temp dir `Path.GetTempPath()/ro3emu-…-{Guid}`, delete on dispose + `SqliteConnection.ClearAllPools()`. Bots log in with lowercase accounts, password `"pw"`. `host.RestartAsync()` keeps the data dir for persistence checks.
  - Deterministic time via a fixed-date `TimeProvider` passed as `clock:`; seeded `new Random(seed)`.
  - Serialize shared-resource suites with `[CollectionDefinition("...", DisableParallelization = true)]`.
  - Golden file `Snapshots/dispatcher-admission.txt`: regenerate only intentionally with `RO3EMU_UPDATE_SNAPSHOTS=1`.
  - Traits: `Category=Slow` (benchmarks, real-client export, quest walk), `Category=StoryWalk` (needs `RO3EMU_STORYWALK=1`).
- **PRs:** template sections *What changes / Evidence and limits / Validation*. List exact commands run and whether exported data and a client were available. Passing CI does not prove the game is playable — say what you could not check.

## Commit on Completion
- When a task is finished and its checks pass, make exactly one `git commit` for that task's changes, then report the commit hash and the full commit message in the final reply.
- Stage only files you changed for the task (`git add <paths>`, never `git add -A`); leave unrelated working-tree changes untouched. Never stage the client, DLLs, `data/`, `re/` or logs.
- Message: Conventional Commits 1.0 (rules below are complete; do not fetch any spec) — `<type>(<scope>): <description>`. Types: `feat`, `fix`, `docs`, `refactor`, `perf`, `test`, `build`, `ci`, `chore`, `revert`. Scope = system/project (`guild`, `world`, `patcher`, `tests`), optional. Description imperative, lowercase, no trailing period, header ≤72 chars (`feat(guild): add bank withdraw limit`). Then a blank line and a short body on what/why and what was not verified; breaking change → `!` after type/scope plus a `BREAKING CHANGE:` footer. No `--amend`, no push, no `--no-verify` unless the user asks.
- If checks fail or the work is incomplete, do not commit; say why.
