# 3D Render Collision Detection

[![C++](https://img.shields.io/badge/C++-17-00599C?style=flat&logo=cplusplus)](https://isocpp.org/)
[![OpenGL](https://img.shields.io/badge/OpenGL-4.6-5586A4?style=flat&logo=opengl)](https://www.opengl.org/)
[![CMake](https://img.shields.io/badge/CMake-Build-064F8C?style=flat&logo=cmake)](https://cmake.org/)

> A real-time 3D renderer focused on collision detection between dynamic ball objects, featuring AABB boundary collision, `DrawBall` per-object draw management, Phong shading, and an ImGui control panel — built as the direct predecessor to the AI Engine project, introducing the patterns (`DrawBall`, `model_data.h`) that carry forward into the full agent simulation.

## Introduction

3D Render Collision Detection is an **interactive physics sandbox** that renders multiple ball objects and resolves their collisions with scene boundaries in real time. It addresses the core challenge of **coupling rigid-body collision response with the OpenGL render loop**, ensuring each frame reflects the physically correct positions of all objects without visible jitter or tunnelling.

Built as a stepping stone between basic 3D rendering (`3D_rotation`) and the full AI Engine, this project introduces per-object draw management (`DrawBall`) and pre-baked model data (`model_data.h`) — patterns that carry forward into all subsequent projects.

## Table of Contents

- [Getting Started](#getting-started)
  - [System Requirements](#system-requirements)
  - [Prerequisites](#prerequisites)
  - [Quick Start](#quick-start)
  - [Manual Build](#manual-build)
  - [Troubleshooting](#troubleshooting)
- [Key Features](#key-features)
- [Architecture](#architecture)
- [Collision System](#collision-system)
- [Design Decisions & Trade-offs](#design-decisions--trade-offs)
- [Project Layout](#project-layout)
- [License](#license)

## Getting Started

### System Requirements

* **OS:** Windows 10/11 (x64)
* **GPU:** OpenGL 4.6-capable GPU (NVIDIA GTX 1050+ or AMD RX 570+ recommended)
* **RAM:** 4 GB minimum
* **Disk:** 500 MB free space

### Prerequisites

* **CMake** 3.15+
* **Visual Studio 2019/2022** with C++ Desktop workload
* **GLEW** and **GLFW3** — pre-built DLLs included in `build/Release/`

### Quick Start

1. **Clone the repository**

   ```bash
   git clone https://github.com/Yupoer/3Drender_Collision_detection.git
   cd 3Drender_Collision_detection
   ```

2. **Configure and build**

   ```bash
   mkdir build && cd build
   cmake ..
   cmake --build . --config Release
   ```

3. **Run**

   ```bash
   .\Release\3DRender.exe
   ```

   > **Note:** Shader files (`fragmentShaderSource.frag`, `vertexShaderSource.vert`) and `picSource/` textures must reside in the same directory as the executable. `glew32.dll` and `glfw3.dll` are pre-placed in `build/Release/`.

### Manual Build

Open `build/3DRender.sln` in Visual Studio and build the `3DRender` target in **Release** configuration.

### Troubleshooting

1. **Balls passing through walls:** Verify the scene AABB dimensions in `main.cpp` match the room geometry. High velocities may require a smaller time step — consider clamping `deltaTime`.
2. **Balls all moving in one direction:** Check that initial velocities are assigned with random direction vectors; a seeded `std::mt19937` is recommended for reproducibility.
3. **Missing DLL error:** Copy `glew32.dll` and `glfw3.dll` from `build/Release/` to the same folder as the executable.
4. **Black screen:** Confirm shader files are co-located with the `.exe` and GPU drivers support OpenGL 4.6.

## Key Features

* **AABB Boundary Collision**: Axis-Aligned Bounding Box tests detect when a ball reaches the scene boundary and reflect its velocity component — preventing tunnelling at high speeds.
* **DrawBall Geometry Manager**: `DrawBall.cpp` encapsulates VAO/VBO setup, uniform binding, and draw calls per ball instance, keeping `main.cpp` free of low-level OpenGL state management and making it trivial to add/remove balls at runtime.
* **Pre-baked Model Data**: `model_data.h` provides vertex arrays generated offline from STL geometry by `stl2array.exe`, enabling zero-latency model loading at startup.
* **Phong Lighting**: Per-fragment ambient, diffuse, and specular shading in GLSL applied to all ball instances with a shared light source.
* **ImGui Control Panel**: Real-time adjustment of ball count, speed, and scene parameters via an integrated **Dear ImGui** overlay.
* **STL Converter Tools**: `stl2array.exe` and `stl_to_vertex_array.cpp` allow regenerating `model_data.h` from updated STL source geometry.
* **stb_image Texturing**: Single-header texture loader applies `.jpg` surface textures to scene geometry.

## Architecture

The system separates physics, rendering, and geometry into independent concerns:

```
main.cpp  (Simulation + Render Loop)
  ├── DrawBall    — per-ball VAO & draw call manager
  ├── AABB.h      — boundary collision detection
  ├── Shader      — GLSL shader loader
  ├── Camera      — view + projection matrices
  ├── Dear ImGui  — runtime control panel
  └── model_data.h — pre-baked ball geometry
```

### Per-Frame Loop

**Physics Update:**
1. Integrate velocity → update each ball's world position
2. **AABB** boundary test — if any axis exceeds the scene box, negate that velocity component
3. Update model matrix for each ball

**Render Pass:**
1. Clear colour + depth buffers
2. For each ball: `DrawBall::Draw()` → bind VAO, set MVP + lighting uniforms, `glDrawArrays`
3. Render ImGui overlay

**Why this structure?**
- **DrawBall encapsulation:** Isolating VAO management and draw calls per object makes adding or removing balls at runtime trivial — just push/pop from the ball vector.
- **AABB for boundaries:** Wall collision is axis-aligned by definition — AABB is the exact correct primitive, not an approximation.

## Collision System

The collision system uses **AABB boundary detection** with **velocity reflection**:

| Collision Type | Method | Resolution |
|----------------|--------|------------|
| **Ball vs Wall (X axis)** | `ball.pos.x ± radius > wall.x` | `velocity.x *= -1` |
| **Ball vs Wall (Y axis)** | `ball.pos.y ± radius > wall.y` | `velocity.y *= -1` |
| **Ball vs Wall (Z axis)** | `ball.pos.z ± radius > wall.z` | `velocity.z *= -1` |

**Why velocity reflection over position clamping?**
Clamping position produces a visual snap that is jarring at high speeds. Reflecting velocity maintains momentum and produces physically plausible bouncing without discontinuities.

**Note:** Ball-to-ball collision is intentionally excluded — this project focuses on environment boundary collision. The AI Engine (the successor project) introduces bounding-sphere agent-agent collision on top of this foundation.

## Design Decisions & Trade-offs

* **Why `DrawBall` instead of raw draw calls in `main`?** Centralising OpenGL state (VAO bind, uniform set, draw call) in `DrawBall` prevents state-leak bugs when rendering multiple instances and makes the per-ball interface consistent across future extensions (e.g., adding AI behaviour in the AI Engine fork).
* **Why pre-bake geometry into `model_data.h`?** File I/O at startup introduces platform-specific path handling and error cases. Embedding geometry as a `const float[]` eliminates both, at the cost of a larger binary — acceptable for a single ball model used across multiple instances.
* **Why AABB-only collision (no OBB or sphere-sphere)?** Ball-to-ball collision is handled at the AI Engine level. This project focuses on environment collision — where AABB is the exact correct primitive for an axis-aligned rectangular room.
* **Why `stb_image` over a dedicated texture library?** `stb_image.h` is a single-file, zero-dependency header that covers all common texture formats. For a project with 2–3 textures, a full texture library would add build complexity with no benefit.

## Project Layout

```plaintext
.
├── AABB.h                       # Axis-Aligned Bounding Box (boundary collision)
├── Camera.cpp / .h              # FPS-style camera controller
├── DrawBall.cpp / .h            # Per-ball VAO, uniform, and draw call manager
├── Shader.cpp / .h              # GLSL shader loader & compiler
├── main.cpp                     # Application entry, physics update, render loop
├── model_data.h                 # Pre-baked ball vertex arrays (from STL)
├── fragmentShaderSource.frag    # Fragment shader (Phong lighting + texturing)
├── vertexShaderSource.vert      # Vertex shader (MVP transform)
├── stb_image.h                  # Single-header texture loader
├── stl2array.exe                # STL-to-vertex-array converter
├── stl_to_vertex_array.cpp      # Converter source code
├── imgui/                       # Dear ImGui (GLFW + OpenGL3 backend)
├── picSource/                   # Texture images (.jpg)
├── build/                       # CMake build output (VS solution + Release exe)
└── CMakeLists.txt               # Build system configuration
```

## License

Distributed under the MIT License. See [LICENSE](LICENSE) for more information.
