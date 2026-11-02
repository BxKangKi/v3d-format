# V3D - Portable 3D Asset, Archive & Terrain Shading Framework

[![Standard](https://img.shields.io/badge/c%2B%2B-17-blue.svg)](https://en.wikipedia.org/wiki/C%2B%2B17)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Shader Model](https://img.shields.io/badge/HLSL-SM%205.0%2B-orange.svg)](shaders/v3d.hlsl)

**V3D** is a lightweight, high-performance 3D asset serialization and archive container format designed for modern simulators, game engines, and graphics pipelines.

The repository provides two core components:
1. **`v3d.hpp`**: A single-header C++17 library for encoding, decoding, validating, and streaming 3D meshes, skeletons, materials, textures, scene metadata, and multi-asset `.v3d` archives.
2. **`v3d.hlsl` / `v3d.ush`**: An engine-independent HLSL shader library providing a robust, derivative-consistent 4-layer terrain shading contract for Unreal Engine and custom DirectX/Vulkan renderers.

---

## Key Highlights

- **Zero Third-Party Dependencies**: The C++ library (`v3d.hpp`) relies solely on the C++17 standard library for all uncompressed operations. An optional decompression callback hook is provided for zlib-compressed archives.
- **Deterministic Binary Encoding**: Explicit little-endian byte ordering, IEEE-754 standard floating-point representation, and length-prefixed UTF-8 strings. No raw C++ struct dumps or compiler padding dependencies.
- **Authored Simulation Coordinates**: Standardized left-handed Z-up coordinates ($+X$ forward, $+Y$ right, $+Z$ up) measured in centimeters. Quaternions in $(X, Y, Z, W)$ order; scale-rotate-translate (TRS) transform convention.
- **4KB-Aligned Streaming Archive**: Revision 6 archive architecture with 4096-byte alignment, 64-byte fixed headers, CRC32 integrity validation, and separate directory/metadata tables.
- **Comprehensive Texture Compression Support**: First-class support for BC1–BC7 (Desktop), ASTC 4x4–12x12 (Mobile), ETC2, and uncompressed float/integer formats.
- **Derivative-Consistent Terrain Shading**: 4-slice `Texture2DArray` blending using explicit `SampleGrad` derivatives to eliminate mip divergence across layer boundaries.

---

## Repository Structure

```
.
├── CMakeLists.txt              # Top-level CMake build configuration
├── LICENSE                     # MIT License
├── README.md                   # Project documentation and quick start
├── docs/
│   └── V3D_FORMAT.md           # Complete binary format and archive specification
├── include/
│   ├── v3d.hpp                 # Single-header C++17 library (root include)
│   └── v3d/
│       └── v3d.hpp             # Canonical namespaced include path
└── shaders/
    ├── v3d.hlsl                # HLSL terrain shading contract (SM 5.0+)
    └── v3d.ush                 # Unreal Engine shader include alias
```

---

## Quick Start: C++ API

### 1. Single-Header Integration
Include `v3d.hpp` directly into your C++17 project:

```cpp
#include "v3d.hpp"
// or if using CMake target: #include <v3d/v3d.hpp>
```

### 2. Creating and Encoding a Mesh
```cpp
#include <v3d/v3d.hpp>
#include <iostream>

void CreateSimpleTriangle() {
    v3d::mesh m;
    m.mesh_index = 0;
    m.name = "TerrainPatch_01";

    v3d::primitive p;
    p.positions = { {0.0f, 0.0f, 0.0f}, {100.0f, 0.0f, 0.0f}, {0.0f, 100.0f, 0.0f} };
    p.normals   = { {0.0f, 0.0f, 1.0f}, {0.0f, 0.0f, 1.0f}, {0.0f, 0.0f, 1.0f} };
    p.uvs.push_back({ {0.0f, 0.0f}, {1.0f, 0.0f}, {0.0f, 1.0f} });
    p.indices = {0, 1, 2};
    p.has_indices = true;

    m.primitives.push_back(std::move(p));

    std::string error;
    v3d::bytes encoded;
    if (!v3d::encode(m, encoded, error)) {
        std::cerr << "Encoding failed: " << error << "\n";
        return;
    }

    // Decode back
    v3d::mesh decoded;
    if (v3d::decode(encoded, decoded, error)) {
        std::cout << "Successfully decoded mesh: " << decoded.name << "\n";
    }
}
```

### 3. Streaming Archive Authoring & Reading
```cpp
#include <v3d/v3d.hpp>

void WriteAndReadArchive(const std::string& filepath) {
    std::string err;
    v3d::archive_writer writer;
    if (!writer.open_file(filepath, err)) return;

    // 1. Add model payload members
    v3d::mesh m; /* populate mesh */
    v3d::range mesh_range;
    writer.add("models/sample_uuid/mesh_0.dat", m, mesh_range, err);

    // 2. Build model directory record and finish
    v3d::archive_directory dir;
    dir.config_range.name = "config.json";
    // Populate directory metadata...

    v3d::guid build_id{{0x1234, 0x5678, 0x9ABC, 0xDEF0}};
    writer.finish(dir, build_id, err);

    // Read back
    v3d::archive_reader reader;
    if (reader.open_file(filepath, err)) {
        const auto& header = reader.header();
        std::cout << "Opened archive revision: " << header.revision << "\n";
    }
}
```

---

## Quick Start: Terrain Shading (HLSL / Unreal Engine)

The shader contract in `shaders/v3d.hlsl` blends 4 layers of `Texture2DArray` textures using baked vertex RGBA weights:

```hlsl
#include "v3d.hlsl" // Or #include "/Project/Shaders/v3d.ush" in Unreal Engine

// In Pixel Shader / Unreal Custom Node:
float3 NormalTS, ORM, Emissive;
float3 BaseColor = V3D_SampleTerrain(
    BaseColorArray, BaseColorSampler,
    NormalArray, NormalSampler,
    ORMArray, ORMSampler,
    EmissiveArray, EmissiveSampler,
    UV, VertexWeights,
    NormalTS, ORM, Emissive
);
```

### Shading Principles
- **No Weight Boundary Divergence**: Uses `ddx(UV)` and `ddy(UV)` with `SampleGrad` so all array slices sample consistent mip levels regardless of weight discontinuities.
- **Layer Weights**: Vertex color `RGBA` maps directly to Layer 0, 1, 2, 3 weights. Alpha is Layer 3, not transparency.
- **Tangent Space Normals**: Layer normal samples are converted from UNORM to $[-1, 1]$, normalized per-layer, blended by weights, and re-normalized.

---

## CMake Integration

Add V3D to your project using CMake FetchContent or submodule:

```cmake
include(FetchContent)
FetchContent_Declare(
    v3d
    GIT_REPOSITORY https://github.com/your-org/v3d.git
    GIT_TAG        main
)
FetchContent_MakeAvailable(v3d)

target_link_libraries(your_target PRIVATE v3d::v3d)
```

---

## Specification Reference

For the comprehensive technical specification of member FourCC signatures, chunking layouts, LOD arrays, and streaming semantics, see [docs/V3D_FORMAT.md](docs/V3D_FORMAT.md).

---

## License

This project is licensed under the [MIT License](LICENSE) - see the `LICENSE` file for details.

Copyright (c) 2026 BxKangKi.
