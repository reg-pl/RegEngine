# Code Review (scope: `Source/`, excluding `Source/packages/`)

## Summary
I reviewed the in-repo engine code under `Source/` (excluding `Source/packages/`) with focus on logic bugs, crash hazards, indexing errors, leaks, naming/spelling, and high-level structure.

---

## Critical / High-Risk Issues

### 1) Out-of-bounds access in light toggle hotkey
- **Location:** `Source/Game.cpp:143-144`
- **Issue:** Key `'3'` checks `m_Lights.size() > 1` but then accesses `m_Lights[2]`.
- **Why it is risky:** When size is exactly 2, this is out-of-bounds and causes undefined behavior/crash.
- **Suggested fix:** Change guard to `m_Lights.size() > 2`.

### 2) Null pointer dereference possibility when mesh has no UV0
- **Location:** `Source/Renderer.cpp:1316-1317, 1325`
- **Issue:** Validation allows meshes with no texture coordinates (`HasTextureCoords(0)` can be false due to `|| !HasTextureCoords(1)`), but code unconditionally reads `mTextureCoords[0][i]`.
- **Why it is risky:** `mTextureCoords[0]` may be null, causing a crash in release and debug.
- **Suggested fix:** Either require UV0 strictly (`CHECK_BOOL(assimpMesh->HasTextureCoords(0))`) or branch and synthesize fallback UVs (e.g., `vec2(0,0)`) when absent.

### 3) Frame-time monotonicity guard uses previous frame instead of current frame
- **Location:** `Source/Time.cpp:95-99`
- **Issue:** `NewFrameFromNow()` clamps with `max(now-start, m_PreviousTime)`.
- **Why it is risky:** If wall-clock goes slightly backward but stays above `m_PreviousTime`, `newTime` can still be below `m_Time`, producing negative `m_DeltaTime`.
- **Suggested fix:** Clamp against `m_Time` (current frame time) rather than `m_PreviousTime`.

### 4) 64-bit size truncation in file I/O
- **Location:** `Source/Streams.cpp:105-108, 116-119`
- **Issue:** `size_t` byte counts are cast to `DWORD` without bounds checks.
- **Why it is risky:** For buffers/files >4 GiB, operations truncate silently; reads/writes become partial/incorrect, potentially corrupting data flow.
- **Suggested fix:** Loop in chunks (`DWORD` max each iteration), or reject oversize requests explicitly with a descriptive error.

### 5) Potential fixed-array overflow in render-target binding helper
- **Location:** `Source/CommandList.cpp:106-131`
- **Issue:** Writes into local `RTVDescriptors[RENDER_TARGET_MAX_COUNT]` based on `renderTargets.size()` with no explicit upper bound check.
- **Why it is risky:** If caller ever passes >8 render targets, stack corruption occurs.
- **Suggested fix:** Add `CHECK_BOOL(renderTargets.size() <= RENDER_TARGET_MAX_COUNT)` before the loop.

### 6) Ring buffer allows zero-capacity initialization path leading to modulo-by-zero hazard
- **Location:** `Source/MultiFrameRingBuffer.hpp:12-17, 51-55`
- **Issue:** `Init()` does not assert `capacity > 0`; allocation path performs `% m_Capacity`.
- **Why it is risky:** If initialized with capacity 0 and allocation of size 0 occurs, modulo by zero is undefined behavior.
- **Suggested fix:** Enforce `capacity > 0` in `Init()`, and optionally reject zero-size allocations.

---

## Naming / Spelling / Clarity Issues

### Typos in identifiers and comments
- `DivideRoudingUp` typo in function name (`Rouding` vs `Rounding`). (`Source/BaseUtils.hpp:101`)
- `GetInsatnce` typo in API name (`Insatnce` vs `Instance`). (`Source/Main.hpp:44`)
- `m_PritimitveTopology` typo in member name (`Pritimitve` vs `Primitive`). (`Source/CommandList.hpp:48`)
- Comment typo `mouse even` (`event`). (`Source/Game.hpp:12`)
- Comment typo `absolute-cannonical-uppercase` (`canonical`). (`Source/SmallFileCache.cpp:18`)
- Comment typo `Woudln't fit` (`Wouldn't fit`). (`Source/MultiFrameRingBuffer.hpp:105`)

### Why this matters
These are low risk at runtime but increase cognitive load, make APIs harder to discover (autocomplete/search), and reduce maintainability.

---

## High-Level Structure / Design Observations

### 1) Heavy global singleton-style state coupling
- **Examples:** `g_App`, `g_Renderer`, `g_SmallFileCache` globals used across many subsystems.
- **Concern:** Tight coupling and hidden dependencies make testing, lifecycle control, and deterministic teardown harder.
- **Improvement direction:** Move toward explicit dependency injection (constructor parameters or context object), especially for utility classes and systems that currently fetch state globally.

### 2) `Renderer` as a large God-object
- **Observation:** `Renderer` owns GPU resource management, shader compilation, asset/model import flow, camera, scene content, frame submission, and UI-related plumbing.
- **Concern:** Broad ownership and responsibilities increase change risk and hinder focused unit-level testing.
- **Improvement direction:** Split into focused components (e.g., `RenderBackend`, `SceneLoader`, `MaterialSystem`, `FrameGraph/PassManager`, `CameraSystem`) with narrow interfaces.

### 3) Validation strategy depends heavily on assertions
- **Observation:** Several invariants are only asserted (debug-only), while release behavior may proceed into UB (e.g., indexing and pointer assumptions).
- **Concern:** Production builds can fail unpredictably on malformed/edge input.
- **Improvement direction:** Keep assertions, but add runtime guards for externally sourced data (asset loading, dynamic arrays, descriptor counts).

---

## Review Notes
- Scope honored: only `Source/` was reviewed, excluding `Source/packages/` and `ThirdParty/`.
- This document focuses on defects and maintainability risks; not all style inconsistencies were enumerated.
