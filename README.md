# Pulse — Windows FPS Booster & Rendszeroptimalizáló

Prémium, sötét témájú WPF alkalmazás Windows 10/11 x64 rendszerekhez.
**.NET 8**, **0 NuGet-függőség**, rendszergazdai jogot kérő `app.manifest`.

> **Figyelmeztetés:** a program rendszerszintű registry- és szolgáltatásmódosításokat végez.
> Használat előtt készíts biztonsági mentést (a program automatikusan készít snapshotot),
> és érdemes egy Windows rendszer-visszaállítási pontot is létrehoznod.

---

## Projektszerkezet

```
Pulse/
├─ Pulse.csproj          # net8.0-windows, WPF, single-file publish
├─ app.manifest               # requireAdministrator + PerMonitorV2 DPI
├─ App.xaml / App.xaml.cs     # belépés, admin-check, globális kivételkezelés
├─ MainWindow.xaml / .cs      # UI: navigáció, kapcsolók, élő napló, telemetria
├─ Themes/
│  └─ Dark.xaml               # teljes dark téma (ecsetek + control template-ek)
├─ Models/
│  ├─ LogEntry.cs             # naplórekord, állapotok, BoostSummary
│  ├─ SettingItem.cs          # egy kapcsoló a felületen (INotifyPropertyChanged)
│  └─ AppConfig.cs            # %ProgramData%\Pulse\config.json
├─ Core/
│  ├─ RegToggle.cs            # RegistryTuner + RegistryCatalog + snapshot/restore
│  ├─ ProcessRunner.cs        # rejtett, időkorlátos powercfg/sc/reg futtatás
│  ├─ AppLogger.cs            # valós idejű + fájl naplózás
│  ├─ NativeMethods.cs        # Win32 P/Invoke (EmptyWorkingSet, ntdll, token…)
│  └─ OptimizerEngine.cs      # facade: kapcsolók, boost, visszaállítás, felmérés
└─ Services/
   ├─ BackupManager.cs        # snapshot.json + reg export/import + visszaállítás
   ├─ PowerPlanService.cs     # Ultimate Performance terv + powercfg hangolás
   ├─ WindowsServiceManager.cs# SysMain / DiagTrack / dmwappush / WSearch / Xbox
   ├─ PerformanceOptimizer.cs # GPU priority, PowerThrottling, GameMode, core parking, HAGS
   ├─ NetworkOptimizer.cs     # Nagle tiltás csatolónként + NetworkThrottlingIndex
   ├─ VisualEffectsService.cs # „Legjobb teljesítmény” effektek
   ├─ CleanupService.cs       # temp/gyorsítótár törlés, Lomtár
   ├─ MemoryOptimizer.cs      # EmptyWorkingSet + standby list purge
   └─ TelemetryService.cs     # CPU / GPU / RAM / előtérben lévő folyamat
```

## Funkciók

| # | Funkció | Megvalósítás |
|---|---------|--------------|
| 1 | Biztonsági mentés és visszaállítás | `BackupManager` — `snapshot.json` (értékszintű) + `registry\*.reg` (reg export) + `services.json` + `power.json` |
| 2 | Teljesítmény és CPU | Ultimate Power Plan, GPU Priority=8 a Games profilban, `PowerThrottlingOff=1`, Game Mode, Core Parking (`ValueMin=100` + `CPMINCORES 100`), `Win32PrioritySeparation=0x26`, HAGS |
| 3 | Szolgáltatások | `sc stop` + `sc config … start= disabled` — SysMain, DiagTrack, dmwappushservice; opcionálisan WSearch és Xbox |
| 4 | Hálózat / latency | `TcpAckFrequency=1`, `TCPNoDelay=1` minden csatolón; `NetworkThrottlingIndex=0xFFFFFFFF`, `SystemResponsiveness=10` |
| 5 | Tisztítás / RAM | `%TEMP%`, `C:\Windows\Temp`, WU-cache, Prefetch, shader cache, Lomtár; `EmptyWorkingSet` + standby list purge |
| 6 | Vizuális effektek | `VisualFXSetting=2`, `UserPreferencesMask`, animációk, átlátszóság, Aero Peek kikapcsolva |

## Fordítás

```
# 1) .NET 8 SDK szükséges
dotnet --list-sdks

# 2) build
dotnet build -c Release

# 3) egyetlen, self-contained exe
dotnet publish -c Release -r win-x64 --self-contained true `
  -p:PublishSingleFile=true -p:IncludeNativeLibrariesForSelfExtract=true `
  -p:EnableCompressionInSingleFile=true
```

A kész exe: `bin\Release\net8.0-windows\win-x64\publish\Pulse.exe`
Indításkor a manifest miatt automatikusan UAC-prompt jelenik meg.

## Fájlok, amiket a program ír

- `%ProgramData%\Pulse\config.json` — kapcsoló-állapotok, eredeti szolgáltatás-típusok
- `%ProgramData%\Pulse\Backups\<dátum>\` — snapshot.json, registry\*.reg, services.json, power.json, meta.json
- `%ProgramData%\Pulse\logs\pulse-<dátum>.log` — futási napló

## Kézi visszaállítás (ha a program nem indul)

```
# a legutóbbi mentés mappa
dir "%ProgramData%\Pulse\Backups"

# értékek visszaállítása (rendszergazdai parancssorban)
reg import "%ProgramData%\Pulse\Backups\<dátum>\registry\power.reg" /reg:64
sc config SysMain start= auto
sc config DiagTrack start= auto
powercfg -setactive 381b4222-f694-41f0-9685-ff5bb260df2e
```

## Ismert korlátok

- A **HAGS** (`HwSchMode=2`) újraindítás után lép életbe, és régebbi GPU-kon nem támogatott.
- A **Game DVR** kikapcsolása letiltja az Xbox Game Bar háttérfelvételét.
- A **WSearch / Xbox** szolgáltatások alapértelmezetten ki vannak kapcsolva a listában (kockázatos csoport),
  és bekapcsolásukkor a program megerősítést kér.
- **GPU-számlálók** hiányozhatnak virtualizált környezetben — ilyenkor a GPU mező 0%-ot mutat.
- Vállalati környezetben a DiagTrack letiltása ütközhet csoportházirenddel.

## Licenc

MIT — szabadon módosítható és terjeszthető. A használatból eredő károkért
a szerző felelősséget nem vállal.