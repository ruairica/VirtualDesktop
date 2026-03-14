# VirtualDesktopTools

A pair of Windows 11 utilities for working with virtual desktops. Originally forked from [shanselman/MaximizeToVirtualDesktop](https://github.com/shanselman/MaximizeToVirtualDesktop); the original maximize-to-desktop functionality has since been removed in favour of the two tools below.

## DeskSwitch

`Ctrl+Alt+Space` overlay to fuzzy-search, switch, create, rename, and remove virtual desktops.

### Usage

| Key | Action |
|-----|--------|
| `Ctrl+Alt+Space` | Open/close overlay |
| Type to filter | Fuzzy-search desktops by name |
| `Enter` | Switch to selected desktop |
| `+name` + `Enter` | Create a new desktop named `name` |
| `F2` | Rename selected desktop |
| `Delete` | Remove selected desktop |
| `Escape` | Close overlay |

### Build & Install

1. Publish the single-file binary:

```sh
cd src/DeskSwitch && dotnet publish -c Release -r win-x64 --self-contained
```

2. Add to Windows startup (PowerShell):

```powershell
$ws = New-Object -ComObject WScript.Shell; $sc = $ws.CreateShortcut("$env:APPDATA\Microsoft\Windows\Start Menu\Programs\Startup\DeskSwitch.lnk"); $sc.TargetPath = "FULL_PATH_TO_REPO\src\DeskSwitch\bin\Release\net10.0-windows\win-x64\publish\DeskSwitch.exe"; $sc.Save()
```

Replace `FULL_PATH_TO_REPO` with the path to your cloned repo.

---

## DevDesk

Creates a new named virtual desktop and launches Windows Terminal + VS Code + Chrome pre-arranged (Terminal left, VS Code right, Chrome behind). Personalised to a specific workflow.

### Build

```sh
cd src/DevDesk && dotnet publish -c Release -r win-x64 --self-contained
```

---

## Requirements

- **Windows 11** (build 22000+)
- Self-contained — no .NET installation required

## Architecture

```
src/
├── WindowsVirtualDesktop.Interop/   Shared COM interop library
│   ├── VirtualDesktopService.cs     High-level virtual desktop operations
│   ├── VirtualDesktopCom.cs         Vendored COM interfaces (from MScholtes/VirtualDesktop)
│   ├── DesktopManagerAdapter.cs     Multi-version COM vtable adapter (pre-24H2 and 24H2+)
│   └── NativeMethods.cs             P/Invoke declarations
├── DeskSwitch/                      WPF overlay app
│   ├── App.xaml / App.xaml.cs       Hotkey registration, lifecycle
│   ├── MainWindow.xaml / .cs        Search UI, desktop operations
│   └── FuzzyMatcher.cs              Fuzzy search scoring
└── DevDesk/                         Console app
    ├── Program.cs                   Desktop creation, app launching, window detection
    └── WindowSnapper.cs             Window positioning helpers
```

**Zero NuGet dependencies.** COM interop declarations are vendored from [MScholtes/VirtualDesktop](https://github.com/MScholtes/VirtualDesktop) (MIT license, actively maintained).

## The Virtual Desktop GUID Problem

Microsoft's Virtual Desktop feature has a proper, documented COM interface — `IVirtualDesktopManager` — but it can only tell you *which* desktop a window is on and move a window *you own* between desktops. The actually useful operations — creating desktops, switching desktops, moving *any* window, naming desktops — all live behind **undocumented COM interfaces** like `IVirtualDesktopManagerInternal` and `IVirtualDesktop`.

The problem? **Microsoft changes the interface GUIDs with nearly every major Windows update.** Not the methods. Not the signatures. Just the GUIDs. This means every app that uses virtual desktop automation — [Peach](https://peachapp.net), [FancyWM](https://github.com/FancyWM/fancywm), and dozens of others — breaks silently 2-3 times a year and has to scramble to update hardcoded GUIDs.

This is the single biggest fragility in these apps. When it breaks, you'll see "Failed to initialize Virtual Desktop COM interface." at startup. The fix is straightforward but shouldn't be necessary:

### How to update the GUIDs

1. Check [MScholtes/VirtualDesktop](https://github.com/MScholtes/VirtualDesktop) — Markus Scholtes maintains per-build interface files (e.g., `VirtualDesktop11-24H2.cs`) and typically updates within days of a new Windows build.
2. If it's a **vtable change** (new/removed methods, same GUIDs), add a new adapter class in `DesktopManagerAdapter.cs` and a new COM interface in `VirtualDesktopCom.cs`.
3. If it's a **GUID change**, update the GUIDs on the affected interfaces.
4. The fragile GUIDs are on these interfaces:

| Interface | What it does | Stable? |
|-----------|-------------|---------|
| `IVirtualDesktopManager` | Check/move owned windows | ✅ Documented, stable since Win10 |
| `IServiceProvider10` | Standard COM service lookup | ✅ Stable |
| `IObjectArray` | Standard COM collection | ✅ Stable |
| `IVirtualDesktop` | Desktop identity, name, wallpaper | ⚠️ **Breaks with Windows updates** |
| `IVirtualDesktopManagerInternal` | Create, switch, move, remove desktops | ⚠️ **Breaks with Windows updates** |
| `IApplicationView` | Window view for cross-process moves | ⚠️ **Breaks with Windows updates** |
| `IApplicationViewCollection` | Get views by window handle | ⚠️ **Breaks with Windows updates** |

The app ships two vtable layouts (pre-24H2 and 24H2+) and auto-selects the correct one at startup with a smoke test fallback.

### Dear Microsoft

Please stabilize the Virtual Desktop COM interfaces or provide a proper public API. Every third-party virtual desktop tool in the ecosystem depends on reverse-engineered GUIDs that break with every update. [PowerToys has asked for this too](https://github.com/microsoft/PowerToys/issues/13993).

## Credits

- **[Markus Scholtes (MScholtes/VirtualDesktop)](https://github.com/MScholtes/VirtualDesktop)** — the COM interface definitions we vendor are from his project (MIT license). He does the hard work of reverse-engineering and publishing updated GUIDs for every Windows build.
- Originally forked from [shanselman/MaximizeToVirtualDesktop](https://github.com/shanselman/MaximizeToVirtualDesktop).

## License

MIT

