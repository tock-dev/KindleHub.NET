# KindleHub.NET
Native .NET client for KindleHub Pro. The default connection uses the same-site API route used by the current web application. See [TESTING.md](TESTING.md) for verified features, automated checks, and remaining limitations.

Cross-platform KindleHub client with a VB.NET core library and Avalonia UI frontend. Desktop builds run on Windows, Linux, and macOS; an Android mobile host is in its initial port phase. (The iOS host was removed.)

## Architecture

- **KindleHub.Core** (VB.NET, .NET 8.0) - Core library with:
  - Authentication (SHA-256 + AES-GCM; password-derived KHA2 envelopes, including recovery metadata)
  - Supabase REST API client
  - Domain models (UserProfile, Group, Message, LeaderboardEntry, AppInfo, etc.)

- **KindleHub.Client.Shared** (C#, Avalonia UI, .NET 8.0) - Shared views, view models, assets, and dependency setup used by each host.
- **KindleHub.Client** (C#, Avalonia desktop host, .NET 8.0) - Windows, macOS, and Linux launcher with:
  - MVVM architecture
  - Views: Home, Leaderboards, Community, Messages, App Store, Settings
  - Dependency injection with Microsoft.Extensions.Hosting
- **KindleHub.Client.Android** - .NET 10 platform launcher using the shared UI and a touch-sized bottom navigation bar.

## Prerequisites

- **.NET 10.0.100 SDK** or later ([download](https://dotnet.microsoft.com/download)). The app still targets .NET 8; the newer compiler is required by Avalonia 12.1's XAML analyzers.
- **Git** (for cloning)

### Platform-specific notes

| Platform | Notes |
|----------|-------|
| **Windows** | Use PowerShell script (`build.ps1`). Requires Windows 10/11 x64 or ARM64. Untested! |
| **Linux** | Use bash script (`build.sh`). Supports X11 and native Wayland sessions. Tested on Debian 13+ (KDE Plasma 6.7.4, Wayland), EndavourOS (KDE Plasma 6.7.4, Wayland) |
| **macOS** | Use bash script (`build.sh`). Requires macOS 12+ (Monterey). Release builds include separate Intel (`osx-x64`) and Apple Silicon (`osx-arm64`) packages. |
| **Android** | Install the .NET Android workload, Android SDK Platform 36, Android build tools, and JDK 21. Build with `dotnet build KindleHub.Client.Android/KindleHub.Client.Android.csproj`. |

The mobile shell reuses the existing views and view models. Actions that open standalone desktop windows (such as story/game viewers and message action panels), and launching downloaded HTML apps, still need mobile page/dialog or platform-native replacements.

## Quick Start

### Linux / macOS

```bash
# Clone and navigate
cd KindleHubClient

# Make script executable (first time only)
chmod +x build.sh

# Build (Release configuration)
./build.sh

# Build and publish self-contained
./build.sh Release --publish --self-contained

# Run published app
./artifacts/Release/linux/KindleHub    # Linux
./artifacts/Release/darwin/KindleHub   # macOS
```

GitHub release packages are published as `...-osx-x64.tar.gz` for Intel Macs
and `...-osx-arm64.tar.gz` for Apple Silicon Macs. Use the `osx-x64` RID when
building locally for an Intel Mac. Automated releases also include an
installable Android arm64 APK (`...-android-arm64.apk`).

### Windows (PowerShell)

```powershell
# Navigate to project
cd KindleHubClient

# Build (Release configuration)
.\build.ps1

# Build and publish self-contained
.\build.ps1 -Configuration Release -Publish -SelfContained -Runtime win-x64

# Run published app
.\artifacts\Release\windows\KindleHub.bat
```

## Build Script Options

### Linux/macOS (`build.sh`)

```bash
./build.sh [Configuration] [Options]

Configuration:  Debug | Release (default: Release)

Options:
  --publish          Publish the application after building
  --self-contained   Include .NET runtime (no .NET install required on target)
  --runtime <RID>    Target runtime identifier (default: auto-detect)
```

**Common Runtime Identifiers (RIDs):**
- `linux-x64`, `linux-arm64`
- `osx-x64`, `osx-arm64`
- `win-x64`, `win-arm64`

### Windows (`build.ps1`)

```powershell
.\build.ps1 [-Configuration <Debug|Release>] [-Publish] [-SelfContained] [-Runtime <RID>]

Parameters:
  -Configuration    Debug | Release (default: Release)
  -Publish          Publish after building
  -SelfContained    Include .NET runtime
  -Runtime          Target RID (required for -SelfContained, default: win-x64)
```

## Development Build

For development with hot reload:

```bash
# Linux/macOS
cd KindleHubClient/KindleHub.Client
dotnet watch run

# Windows (PowerShell)
cd KindleHubClient/KindleHub.Client
dotnet watch run
```

## Project Structure

```
KindleHubClient/
├── KindleHub.sln                 # Solution file
├── build.sh                      # Linux/macOS build script
├── build.ps1                     # Windows build script
├── KindleHub.Core/               # VB.NET core library
│   ├── KindleHub.Core.vbproj
│   ├── Api/                      # Supabase REST API client
│   ├── Crypto/                   # Encryption (PBKDF2, AES-GCM)
│   ├── Models/                   # Domain models
│   └── Services/                 # Core services
└── KindleHub.Client/             # Avalonia UI frontend
    ├── KindleHub.Client.csproj
    ├── ViewModels/               # MVVM ViewModels
    ├── Views/                    # XAML Views
    ├── Services/                 # UI services
    ├── Program.cs                # Entry point + DI setup
    └── App.axaml                 # Application resources
```

## Output Locations

| Build Type | Location |
|------------|----------|
| Standard build | `KindleHub.Core/bin/<Config>/net8.0/`, `KindleHub.Client/bin/<Config>/net8.0/` |
| Published (Linux) | `artifacts/<Config>/linux/` |
| Published (macOS) | `artifacts/<Config>/darwin/` |
| Published (Windows) | `artifacts/<Config>/windows/` |

## Troubleshooting

### "GTK not found" on Linux
```bash
# Ubuntu/Debian
sudo apt install libgtk-3-0

# Fedora
sudo dnf install gtk3

# Arch
sudo pacman -S gtk3
```

### "Avalonia XAML compilation errors"
- Ensure `EnableAvaloniaXamlCompilation=true` in `.csproj`
- Avoid `x:DataType` in XAML (uses compiled bindings without it)
- Use `PlaceholderText` instead of `Watermark` on TextBox

### Build fails with "namespace not found" (VB.NET)
- Remove explicit `Namespace` blocks from `.vb` files
- Use `RootNamespace` in `.vbproj` instead
- Ensure `Imports System.Threading` and `Imports System.Collections.Generic` are present

### Self-contained publish is large
- **Windows**: Enable trimming (`-p:PublishTrimmed=true`) and single-file (`-p:PublishSingleFile=true`) - default in build script
- **Linux/macOS**: Trimming disabled by default (breaks Avalonia XAML); runtime is included but not trimmed

## API Configuration

The default API base is `https://kindlehub.pro/t/api`, the web application's
same-site tunnel. It serves the existing `/rest/v1/` API. The direct worker
address failed the native .NET reachability test during this repair; the same-site
route passed public feature tests. No browser identity spoofing is required.

For a self-hosted backend, set `KindleHubApiOptions.BaseUrl` when constructing the
core. The base URL must include any route prefix and exclude `/rest/v1/`.

This is an unofficial client: server permissions, moderation, rate limits, and
subscriptions still apply. It does not override server refusals.

## Automated tests

Install the .NET 10 SDK and the .NET 8 runtime (the desktop/game runner targets
.NET 8), then run:

```bash
./test.sh
# Also check the real public service (no writes or credentials):
KINDLEHUB_LIVE_TESTS=1 ./test.sh
# Explicitly create/reuse disposable accounts and test private writes:
KINDLEHUB_LIVE_ACCOUNT_TESTS=1 KINDLEHUB_LIVE_TESTS=1 ./test.sh
```

The script also finds SDKs installed in `~/.dotnet`. For direct CLI commands,
put that directory first in `PATH`. Desktop builds do not require the Android
workload. Release CI runs deterministic API/state tests, headless UI tests, and
the existing game suite before publishing.

## License

GPL License - see LICENSE file for details. (why does the agent keep messing this up, why was it originally MIT)
