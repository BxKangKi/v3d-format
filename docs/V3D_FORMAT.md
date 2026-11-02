# V3D File & Archive Format Specification

The **V3D** specification defines a high-performance, portable binary format for 3D geometric models, scene hierarchies, materials, textures, audio, and archive containers. It is authored specifically for modern simulation and real-time game engines.

---

## 1. Fundamental Conventions

### 1.1 Endianness and Primitive Types
- **Byte Order**: Explicitly **little-endian** across all integer and floating-point types.
- **Floats**: Standard **IEEE-754** single-precision `binary32` (`float`, 4 bytes) and double-precision `binary64` (`double`, 8 bytes).
- **Strings**: Prefixed by an unsigned 32-bit integer length, followed by UTF-8 bytes with **no null terminator**.
- **Memory Limits**: Single members are capped at $2\text{ GiB} - 1$ (`2147483647` bytes). String fields are capped at $64\text{ MiB}$ (and $4096$ bytes for identifier names).

### 1.2 Coordinate Basis and Transforms
- **Coordinate System**: Left-handed Z-up:
  - `+X`: Forward
  - `+Y`: Right
  - `+Z`: Up
- **Unit of Measurement**: Centimeters ($1.0\text{ unit} = 1.0\text{ cm}$).
- **Quaternions**: Component order is $(X, Y, Z, W)$ where $W$ is the scalar real part.
- **Transformation Order**: Scale $\rightarrow$ Rotate $\rightarrow$ Translate:
  $$\mathbf{P}' = \mathbf{T} + \mathbf{R} \cdot (\mathbf{S} \odot \mathbf{P})$$
  Transform components are stored as 10 double-precision IEEE-754 floats ($4 \times \text{quat} + 3 \times \text{trans} + 3 \times \text{scale}$).

---

## 2. Member Payload Formats

Every serialized V3D member begins with a 4-byte little-endian four-character code (FourCC) magic identifier and a 4-byte unsigned version number.

| Member Type | Magic (Hex) | FourCC | Current Version | Description |
| :--- | :--- | :--- | :--- | :--- |
| **Mesh** | `0x4853454d` | `MESH` | `1` | Geometry primitives, vertex streams, morph targets, skin bindings |
| **Skin** | `0x4e494b53` | `SKIN` | `1` | Skeletal hierarchy, bone names, transforms, joint index mapping |
| **Material**| `0x4c54414d` | `MATL` | `1` | Material flags, scalar parameters, vector parameters, texture parameters |
| **Texture** | `0x52584554` | `TEXR` | `2` | Mip chain, format (BC1-7, ASTC, ETC2, uncompressed), layers, sampling modes |
| **Manifest**| `0x464e414d` | `MANF` | `1` | Member offset ranges, CRC32 checksums, and model member index mappings |
| **Metadata**| `0x31444d47` | `GMD1` | `1` | Scene node trees, chunking indices, water streams, colliders, lights |
| **Definition**| `0x31464447` | `GDF1` | `1` | Bone aliases and auxiliary JSON definition payload |
| **JSON**    | `0x31434647` | `GCF1` | `1` | Standalone UTF-8 JSON configuration and script definitions |

### 2.1 Texture Compression Formats
Texture format codes represent native GPU-friendly layouts:
- `rgba8_unorm` (1), `bgra8_unorm` (2), `r8_unorm` (3), `rg8_unorm` (4)
- `rgba16_float` (5), `r16_float` (6), `r32_float` (7), `rgba32_float` (8), `r16_unorm` (9)
- Block compression (Desktop): `bc1_unorm` (10), `bc2_unorm` (11), `bc3_unorm` (12), `bc4_unorm` (13), `bc5_unorm` (14), `bc6h_ufloat` (15), `bc7_unorm` (16)
- Block compression (Mobile): `astc_4x4_unorm` (20) through `astc_12x12_unorm` (24)
- Block compression (ETC2): `etc2_rgb8_unorm` (30), `etc2_rgba8_unorm` (31)

---

## 3. Archive Container Format (`.v3d`)

The archive container combines multiple models, sounds, scripts, and configuration blocks into a single streaming binary file.

### 3.1 Container Layout
```
+-------------------------------------------------------------+
| Archive Header (64 bytes)                                   |
+-------------------------------------------------------------+
| Payload Member 1 (Aligned to 4096 bytes)                    |
| Payload Member 2 (Aligned to 4096 bytes)                    |
| ...                                                         |
| Payload Member N                                            |
+-------------------------------------------------------------+
| Directory Block (Offset specified in header)                |
+-------------------------------------------------------------+
```

### 3.2 64-Byte Archive Header Layout
1. `uint64_t magic`: `0x00000a0d1a443356` (V3D Revision 6)
2. `uint32_t header_bytes`: Fixed `64`
3. `uint32_t sound_count`: Number of sound records
4. `uint64_t directory_offset`: File offset to directory block
5. `uint64_t directory_size`: Byte length of directory block
6. `uint32_t directory_crc`: CRC32 of directory bytes
7. `uint32_t model_count`: Number of model records
8. `uint32_t build_id[4]`: 128-bit build GUID
9. `uint64_t revision`: Current revision (`6`)

### 3.3 Member Alignment and Compression
- Each member payload is aligned to $4096\text{ bytes}$ (`archive_alignment`).
- Member compression codec:
  - `0`: Stored (uncompressed; direct streaming).
  - `1`: Deflate/Zlib compression.
- An archive reader can read uncompressed members with zero third-party dependencies, or provide a standard zlib callback (`uncompress2` / `inflate`) for compressed assets.

---

## 4. Terrain Shading Contract (`v3d.hlsl` / `v3d.ush`)

The terrain shading pipeline implements an engine-independent HLSL pixel-stage contract (Shader Model 5.0+):
- **Textures**: 4-slice `Texture2DArray` resources representing:
  1. Base Color (`sRGB` view)
  2. Tangent-Space Normals (`Linear` view, RGB UNORM)
  3. ORM (`Linear` view: R = Occlusion, G = Roughness, B = Metallic)
  4. Emissive (`Linear` view)
- **Vertex Attributes**: Vertex RGBA colors represent **layer weights** $[W_0, W_1, W_2, W_3]$. Alpha is layer 3, not opacity.
- **Sampling Strategy**: Derivative-consistent texture sampling using explicit `SampleGrad(Sampler, UVLayer, ddx(UV), ddy(UV))`. This avoids divergent mip calculation artifacts across weight transition boundaries.
- **Unreal Engine Integration**: The HLSL file is byte-identical to `v3d.ush` and directly usable in Unreal Custom HLSL material nodes.
