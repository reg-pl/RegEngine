# AGENTS.md

## Project overview
- **RegEngine** is a personal **3D graphics engine prototype** for **Windows + Direct3D 12**.
- Main implemented areas: D3D12 initialization, model loading via Assimp, basic deferred shading, Dear ImGui UI, JSON-based settings persistence.
- Status is explicitly **work in progress**.

## Repository layout
- `Source/`: engine and app code (Visual Studio solution/project, renderer/game/app, settings, resource management).
- `WorkingDir/`: runtime data and editable content (JSON settings, shaders, fonts).
- `Doc/`: lightweight user docs and screenshot.
- `ThirdParty/` and `Source/packages/`: vendored dependencies.

## Build/run expectations
- Primary toolchain/environment is **Visual Studio 2022 on Windows** with modern Windows SDK and D3D12-capable GPU.
- Entry point is in `Source/Main.cpp` (Windows app lifecycle + renderer/game init).
- In this Linux-based CI/container, full native build/run is generally not expected.

## Runtime/config model
- Settings are divided into categories in `Source/Settings.hpp`:
  - `Startup` -> loaded once from `WorkingDir/StartupSettings.json`.
  - `Load` -> loaded from `WorkingDir/LoadSettings.json` on startup/refresh.
  - `Runtime` -> loaded from `WorkingDir/RuntimeSettings.json`, saved on exit.
  - `Volatile` -> defaults only, not persisted.
- Shader sources are under `WorkingDir/Shaders/`.

## Useful behavior and controls
- Command-line option documented in `Doc/Command line.txt`:
  - `/AssimpPrint <PATH>`: load file with Assimp, print info, then exit.
- Keyboard shortcuts in `Doc/Keyboard shortcuts.txt` include:
  - `ESC` exit, `Pause` pause/resume scene time.
- In-game camera/light controls are implemented in `Source/Game.cpp` (e.g., WSADQE camera movement while RMB drag is active, hotkeys for toggling ambient/lights).

## Notes for future AI agents
- Prefer small, focused changes in `Source/` and `WorkingDir/`.
- Avoid editing vendored code in `ThirdParty/` and `Source/packages/` unless explicitly requested.
- If changing rendering behavior, check related shader files and matching C++ bindings/constants.
- Keep README and docs aligned when introducing user-facing flags, controls, or setup changes.
