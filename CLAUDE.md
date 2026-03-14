# Projects

- **DevDesk** — Creates a new virtual desktop with Terminal + VS Code + Chrome pre-arranged.
- **DeskSwitch** — Ctrl+Alt+Space overlay to fuzzy-search, switch, create, and remove virtual desktops.
- **WindowsVirtualDesktop.Interop** — Shared COM interop library used by both apps. Contains VirtualDesktopService, NativeMethods, and the vendored COM interface declarations.

# Build

After making changes, publish the single-file binaries:

```sh
cd src/DevDesk && dotnet publish -c Release -r win-x64 --self-contained
cd src/DeskSwitch && dotnet publish -c Release -r win-x64 --self-contained
```

Output:
- `src/DevDesk/bin/Release/net10.0-windows/win-x64/publish/DevDesk.exe`
- `src/DeskSwitch/bin/Release/net10.0-windows/win-x64/publish/DeskSwitch.exe`

# COM Interface Maintenance

`src/WindowsVirtualDesktop.Interop/VirtualDesktopCom.cs` contains undocumented Windows COM GUIDs that change with major Windows updates. When either app breaks after a Windows update:
1. Check [MScholtes/VirtualDesktop](https://github.com/MScholtes/VirtualDesktop) for updated GUIDs
2. Update only `VirtualDesktopCom.cs` — the rest of the code is stable
3. The key interfaces are: `IVirtualDesktop`, `IVirtualDesktopManagerInternal`, `IApplicationView`, `IApplicationViewCollection`
