# Instructions

This file provides guidance when working with code in this repository.

## Project Overview

Fork of [BepInEx.ConfigurationManager](https://github.com/BepInEx/BepInEx.ConfigurationManager) adapted for Lobotomy Corporation's mod loader (LMM) and Harmony 1. Provides an in-game ImGUI settings window (F1) for LMM and BepInEx mods. Targets **net35** (Unity 2017-era Mono runtime).

Upstream remote: `upstream` → BepInEx/BepInEx.ConfigurationManager
Origin: `origin` → open-lobotomy/LobCorp.ConfigurationManager

## Build

```bash
dotnet build
```

Output goes to `bin/net35/`. Dependencies in `lib/` are symlinks to `../lobotomy-corporation-mods/external/LobotomyCorp_Data/Managed/` — the game's managed assemblies plus Harmony must be present there.

Tests live in `LobCorp.ConfigurationManager.Test` (xunit.v3, Moq, AwesomeAssertions). Run with `dotnet test`.

## Architecture

**Entry flow:**
1. `Harmony_Patch.cs` — static initializer creates Harmony instance (`com.lobcorp.configurationmanager`), patches all
2. `Patches/IntroPlayerPatchAwake.cs` — Harmony postfix on `IntroPlayer.Awake()` injects `ConfigManagerBehaviour` MonoBehaviour
3. `Implementations/ConfigManagerBehaviour.cs` — Unity lifecycle wrapper, delegates to `ConfigurationManager`
4. `Implementations/ConfigurationManager.cs` — main UI controller, handles hotkey, window rendering

**Settings discovery (`Implementations/SettingSearcher.cs`):**
- LMM mods register via `Config/LmmConfigRegistration.cs` static API
- Auto-scans `BaseMods/{modId}/config.cfg` files
- Discovers BepInEx plugins via reflection (`Implementations/BepInExInterop.cs`) — no hard dependency

**Configuration model (`Config/`):**
- `LmmConfigFile` — file I/O and parsing for `config.cfg` files
- `LmmConfigEntry<T>` — generic typed entries with change events and auto-save
- `LmmConfigDefinition` — section + key identity
- `AcceptableValueRange<T>` / `AcceptableValueList<T>` — value constraints implementing `IAcceptableValue`

**UI rendering (`Implementations/SettingFieldDrawer.cs`):**
- ImGUI immediate-mode rendering
- Type-specific controls: checkboxes, sliders, dropdowns, color pickers, hotkey capture
- `ConfigurationManagerAttributes` controls display (order, visibility, custom drawers)

**`ConfigurationManagerAttributes` is a copy-paste template, not a referenced API.** Each plugin bundles its own copy of the class and assigns values to instances that get passed as tags to setting descriptions. `SettingEntryBase.SetFromAttributes()` reads these via reflection (`Type.GetProperties`, matching by simple type name — not assembly identity). This fork uses **public auto-properties**; upstream BepInEx.ConfigurationManager uses **public fields**, so upstream's template is not directly compatible — if copying from upstream, convert the fields to auto-properties.

## Key Constraints

- **net35 target**: no LINQ extensions beyond what's available, no `System.ValueTuple`, limited BCL. `LangVersion` is set to `latest` so C# syntax features work but BCL APIs are restricted.
- **RootNamespace and AssemblyName are both `ConfigurationManager`** (not `LobCorp.ConfigurationManager`) — intentionally matches upstream BepInEx.ConfigurationManager. This is a **DLL-name / namespace collision prevention** mechanism only: the identical DLL name stops both from loading simultaneously, and the shared root namespace avoids dual-load conflicts (double UI entries, duplicate `ConfigurationManagerAttributes` processing). This is **not** a public-API-compatibility contract — the fork can freely change its internal shape (e.g. `ConfigurationManagerAttributes` was moved from fields to properties). Do not change `RootNamespace` or `AssemblyName` without accounting for the loader-collision implications.
- **`Harmony_Patch` class name is load-bearing** — every LMM mod must expose an entry type named `Harmony_Patch`. The analyzer package (`LobotomyCorporation.Mods.Analyzers` globalconfig) suppresses S101 and CA1707 repo-wide so this pattern doesn't trip naming rules.
- **Game assembly references are `Private=false`** — none are copied to output since they exist in the game's managed folder at runtime. The `LobotomyCorporation.Mods.Common` PackageReference does copy to output, as it must be deployed alongside the mod.
- **Implicit usings and nullable are disabled.**
- `Microsoft.NETFramework.ReferenceAssemblies` is pulled in implicitly by the SDK for net35 — do not add it to `Directory.Packages.props`.

## CI/CD

- **CI** (`.github/workflows/ci.yml`) — calls the pinned reusable Linux and Windows workflows from `open-lobotomy/.github` on pushes to `main` and pull requests. The Linux workflow runs `dotnet ci --check`; the Windows workflow builds and tests `ConfigurationManager.slnx`.
- **Release** (`.github/workflows/release.yml`) — triggered when a GitHub Release is created. Builds, tests, packages the mod as a zip (matching the `BaseMods/ConfigurationManager/` folder structure), and uploads it as a release asset.
- The reusable workflows authenticate GitHub Packages with the job's `GITHUB_TOKEN`. The repository's jobs require `packages: read`, and each private package must grant this repository read access under its GitHub Actions package settings. Do not add a personal `PACKAGES_TOKEN` to new workflows.
- The committed `nuget.config` contains only NuGet.org and the organization GitHub Packages feed. For local package development, copy it to the ignored `nuget.local.config`, add the local source and an exact `packageSourceMapping` entry for each locally developed `LobotomyCorporation.*` or `OpenLobotomy.*` package so that the local source overrides the broader `github` mapping, and invoke restore with `dotnet restore --configfile nuget.local.config`; do not commit machine-specific feed paths.
- The local CI tool manifest pins `OpenLobotomy.Tooling` `0.1.0-preview.54` and CSharpier `1.3.0`. Run `dotnet tool restore` before `dotnet ci --check`.

NuGet package publishing is planned for v1.0.0 but not yet implemented.

## Analyzers

Global analyzers (`LobotomyCorporation.Mods.Analyzers`, `OpenLobotomy.Standards`) are configured in `Directory.Packages.props`. All Sonar rules run at their default severity — there are no global suppressions in `.editorconfig`. Rule exceptions that apply to all LMM mods (e.g. S101/CA1707 for the `Harmony_Patch` entry point) live in the shared `LobotomyCorporation.Mods.Analyzers` globalconfig, not here. Fix violations rather than suppressing them; if a suppression is truly needed, scope it as narrowly as possible (file-local `#pragma` or per-member `[SuppressMessage]`).
