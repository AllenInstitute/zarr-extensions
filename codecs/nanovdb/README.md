# nanovdb codec

Defines an `array -> bytes` codec that encodes an array chunk as one or more
[NanoVDB](https://www.openvdb.org/documentation/doxygen/NanoVDB_MainPage.html)
grid buffer.

NanoVDB is the linearized, pointer-free serialization of an
[OpenVDB](https://www.openvdb.org/) tree, in which regions holding a single
repeated value are stored once as a tile rather than as voxels. For volumes
that are mostly background (masks, sparse labels, level sets, distance fields,
sparse vector fields), the buffer is both a compact encoding and a spatial
index that a reader can traverse directly. This codec constrains how each
chunk maps to a grid so that the grids of all chunks share one index space and
node lattice; see [Grid coherence across chunks](#grid-coherence-across-chunks).

> This document is a proposed extension. It is licensed under the
> [Creative Commons Attribution 3.0 Unported License](https://creativecommons.org/licenses/by/3.0/).

> [!NOTE]
> **Status: draft**, opened to solicit feedback. The reference implementation
> is in progress; see
> [Interoperability and compatibility](#interoperability-and-compatibility).

## Codec name

The value of the `name` member in the codec object MUST be `nanovdb`.

## Configuration parameters

The `configuration` object is REQUIRED and all of its members are required. It
is a closed set: implementations MUST NOT emit members other than those below.

Decoders MUST take grid geometry and grid type from the buffer rather than from
this configuration, MUST verify both against the array metadata and the members
below, and MUST return an error on mismatch.

The buffer does not record [`tree_config`](#tree_config). A decoder MUST reject
a declared `tree_config` it does not implement, and SHOULD check that the
buffer's node byte spans agree with the declared branching factors wherever a
level is non-empty.

### `dimension_roles`

An array of role tokens, one per dimension of this codec's input, outermost
first. The input is the array chunk after any preceding `array -> array` codec
has been applied. The length MUST equal the input rank; every dimension MUST be
given a role explicitly.

| role          | how this codec encodes the dimension                                                           |
| ------------- | ---------------------------------------------------------------------------------------------- |
| `grid`        | Each index becomes a separate NanoVDB grid within the chunk's buffer.                          |
| `z`, `y`, `x` | A NanoVDB coordinate axis. These three dimensions are the tree's index space.                  |
| `channel`     | Becomes the grid's value type: extent `1` a scalar grid, `3` a `Vec3` grid, `4` a `Vec4` grid. |

The roles MUST appear in this order:

```
[grid … grid]   z   y   x   [channel]
```

Zero or more `grid` dimensions, outermost; then exactly one each of `z`, `y` and
`x`, in that order; then at most one `channel` dimension, which MUST be the
innermost (unit-stride) dimension. Encoders and decoders MUST reject any other
arrangement. A `channel` dimension MUST have extent `1`, `3` or `4`, matching
`grid_type` per [Supported data types](#supported-data-types). No `channel`
dimension is equivalent to a `channel` dimension of extent `1`.

Roles constrain this codec's input, not the array. An array in any other
dimension order reaches it through a preceding
[`transpose`](../transpose/README.md); see
[Interaction with other codecs](#interaction-with-other-codecs). Because
`transpose` defines its output dimension `i` to be its input dimension
`order[i]`, `dimension_roles[i]` describes array dimension `order[i]`, and
`dimension_roles` is then **not** index-aligned with `shape`, `chunk_shape` or
`dimension_names`. OME-Zarr `[t, c, z, y, x]` with `c = 3` requires
`order: [0, 2, 3, 4, 1]`; an `[x, y, z]` array requires `order: [2, 1, 0]`.

A role states only how this codec encodes a dimension, never what it means: a
`grid` dimension need not be time, nor a `channel` dimension vector components.
Meaning belongs in `dimension_names` or a convention above Zarr.

### `grid_type`

The NanoVDB `GridType` of the encoded grid: one of `Float`, `Double`, `Int16`,
`Int32`, `Int64`, `UInt32`, `Mask`, `Fp4`, `Fp8`, `Fp16`, `FpN`, `Vec3f`,
`Vec3d`, `Vec4f`, `Vec4d`. Not derivable from the array's `data_type`: the
quantized types (`Fp4`, `Fp8`, `Fp16`, `FpN`) encode a `float32` array at
reduced precision, and the vector types pair a `float32` or `float64` array
with a `channel` dimension. See
[Supported data types](#supported-data-types).

### `tree_config`

An array of the tree's node branching factors, as the `Log2Dim` of each level
from the root side to the leaf. MUST be `[5, 4, 3]`, giving 8³ leaves, 128³
lower internal nodes and 4096³ upper internal nodes. The leaf extent is
`1 << tree_config[2]` and the lower internal node extent is
`1 << (tree_config[1] + tree_config[2])`.

`[5, 4, 3]` is the only registered configuration, and the only one the released
NanoVDB implementation reads or writes. Branching factors appear in neither the
grid header nor the `nanovdb::io` file header, which records node and tile
_counts_ only. Any future value MUST also have exactly three entries: NanoVDB's
tree depth is not parameterized.

### `index_space`

One of `global` or `chunk_local`: whether each chunk's grid uses the array's
global index coordinates or is rebased to the chunk's own origin. `global` is
REQUIRED for [grid coherence](#grid-coherence-across-chunks) and SHOULD be
used.

### `stats`

One of `none`, `bbox`, `minmax`, `all`: the per-node statistics the grid
carries. `bbox` includes active bounding boxes; `minmax` adds per-node minimum
and maximum values; `all` adds average and standard deviation.

### `lossless`

A boolean indicating whether decoding reproduces the encoded array exactly.
When `true`, `tolerance` MUST be `0` and `grid_type` MUST NOT be a quantized
type. When `false`, `tolerance` MUST be greater than `0` or `grid_type` MUST be
a quantized type.

### `tolerance`

A number ≥ 0: the maximum absolute deviation from the background value at which
the encoder may leave a voxel inactive. `0` permits dropping only voxels
exactly equal to the background. See
[Sparsity and losslessness](#sparsity-and-losslessness).

## Supported chunk shapes

The chunk's rank MUST equal the length of
[`dimension_roles`](#dimension_roles). Each dimension is constrained by its
role:

| role      | chunk extent                                                                                                                        |
| --------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| `grid`    | Any extent. The buffer holds one grid per index, so the grid count is the product of the chunk extents along all `grid` dimensions. |
| `z y x`   | Leaf-aligned, and node-aligned where practical: requirements 2 and 3 of [grid coherence](#grid-coherence-across-chunks).            |
| `channel` | MUST equal the array extent along that dimension (requirement 6).                                                                   |

The innermost spatial dimension is the NanoVDB `x` axis. Writing `t` for the
index along a single `grid` dimension and `c` for the `channel` index:

| `dimension_roles`                | chunk shape           | grid value | grid within buffer | NanoVDB coordinate                                      |
| -------------------------------- | --------------------- | ---------- | ------------------ | ------------------------------------------------------- |
| `["z","y","x"]`                  | `[nz, ny, nx]`        | scalar     | the only grid      | `[k, j, i]` → `(i,j,k)`                                 |
| `["z","y","x","channel"]`, `c=1` | `[nz, ny, nx, 1]`     | scalar     | the only grid      | `[k, j, i, 0]` → `(i,j,k)`                              |
| `["z","y","x","channel"]`, `c=3` | `[nz, ny, nx, 3]`     | `Vec3`     | the only grid      | `[k, j, i, c]` → `(i,j,k)` component `c`                |
| `["grid","z","y","x"]`           | `[nt, nz, ny, nx]`    | scalar     | grid `t`           | `[t, k, j, i]` → `(i,j,k)` in grid `t`                  |
| `["grid","z","y","x","channel"]` | `[nt, nz, ny, nx, c]` | `c`-vector | grid `t`           | `[t, k, j, i, c]` → `(i,j,k)` component `c` in grid `t` |

Encoders and decoders MUST return an error if the roles are not in the required
order, if the rank disagrees with `dimension_roles`, if the `channel` extent
disagrees with `grid_type`, if the number of grids in the buffer disagrees with
the chunk's `grid` extents, or if a grid's index bounding box is inconsistent
with the chunk's position and spatial shape.

These rules apply to the inner chunk shape when this codec is used as the
array-to-bytes codec within the
[`sharding_indexed`](../sharding_indexed/README.md) codec.

## Supported data types

Scalar grids take an input with no `channel` dimension, or a `channel`
dimension of extent `1`:

| Zarr `data_type` | `grid_type`                          | Notes                                         |
| ---------------- | ------------------------------------ | --------------------------------------------- |
| `float32`        | `Float`, `Fp4`, `Fp8`, `Fp16`, `FpN` | Quantized types are lossy                     |
| `float64`        | `Double`                             |                                               |
| `int16`          | `Int16`                              |                                               |
| `int32`          | `Int32`                              |                                               |
| `int64`          | `Int64`                              |                                               |
| `uint32`         | `UInt32`                             |                                               |
| `bool`           | `Mask`                               | Topology only; the value _is_ the active mask |

Vector grids take a `channel` dimension of extent `c`:

| Zarr `data_type` | `c` | `grid_type` |
| ---------------- | --- | ----------- |
| `float32`        | `3` | `Vec3f`     |
| `float64`        | `3` | `Vec3d`     |
| `float32`        | `4` | `Vec4f`     |
| `float64`        | `4` | `Vec4d`     |

A `channel` extent of `2`, or greater than `4`, MUST be rejected. Such an axis
SHOULD be given the `grid` role instead, placing each index on its own grid.

Data types not listed above are supported only through promotion:
implementations MAY promote a narrower data type to a wider `grid_type` (for
example `uint8` or `int8` to `Int16`), but MUST record the `grid_type` actually
written and MUST NOT promote across the scalar/vector boundary. Zarr data types
with no NanoVDB counterpart (complex and variable-length types among them) are
not supported.

Matrix grids, which would need two `channel` dimensions, and integer vector
grids (`Vec3i` and similar) are out of scope.

For vector grids, `fill_value` being a scalar means:

- The grid's background value MUST be the vector all of whose components equal
  `fill_value`. A background with unequal components MUST NOT be written.
- A voxel is active or inactive as a whole. An encoder MUST treat a voxel as
  non-background if _any_ component differs from `fill_value` by more than
  `tolerance`.

> [!NOTE]
> **Open question for review.** `stats: minmax` is undefined for vector grids,
> which have no natural total order. Componentwise minima and maxima are the
> plausible reading; the conservative option is to restrict vector grids to
> `stats: none` or `bbox`.

## Format and algorithm

### Encoded representation

The encoded chunk is a NanoVDB grid buffer: the in-memory layout produced by
NanoVDB's grid builders, written verbatim.

- The buffer MUST be little-endian and MUST begin with a valid NanoVDB grid
  magic number and version header. The magic numbers, version encoding, and
  structure layouts are defined normatively by
  [`nanovdb/NanoVDB.h`](https://github.com/AcademySoftwareFoundation/openvdb/blob/master/nanovdb/nanovdb/NanoVDB.h).
- The buffer MUST NOT be wrapped in the `nanovdb::io` file container, whose
  magic number differs from a raw grid buffer's.
- Every grid's background value MUST equal the array's `fill_value`.

#### Grid enumeration

An input whose [`dimension_roles`](#dimension_roles) contain no `grid`
dimension encodes one grid per chunk. Otherwise the chunk's buffer holds
`mGridCount` grids, laid out as `nanovdb::mergeGrids` produces them:

- The grid count MUST equal the product of the chunk's extents along its `grid`
  dimensions, and MUST be recorded as `mGridCount` in **every** grid's header,
  not only the first.
- Grids MUST be stored consecutively, grid `n + 1` beginning `mGridSize` bytes
  after grid `n`. A reader locates grid `n` by summing the preceding grids'
  `mGridSize`.
- Grid `n` MUST record `mGridIndex` equal to `n`, where `n` is the C-order
  (row-major, last `grid` dimension varying fastest) ravel of the chunk-local
  indices along the `grid` dimensions.
- Readers MUST resolve a grid by index, never by `mGridName`, which is
  unconstrained. Writers MAY set names.

### Grid coherence across chunks

An array using this codec MUST satisfy all of the following. Each is checkable
from the array metadata and chunk buffers alone.

1. **One index space.** Each chunk's grids MUST express voxel coordinates in
   the global index space of this codec's input: the `z`, `y`, `x` indices
   `[k, j, i]` occupy NanoVDB coordinate `(i, j, k)` regardless of which chunk
   holds them. Grids MUST NOT be rebased to a chunk-local origin. A preceding
   [`transpose`](../transpose/README.md) composes a constant permutation onto
   this mapping: the array coordinate is recovered from the NanoVDB coordinate
   by `order`.

2. **Leaf-aligned chunk grid.** The chunk extent along each of the `z`, `y` and
   `x` dimensions, and the chunk grid origin in each of them, MUST be an
   integer multiple of the leaf extent `1 << tree_config[2]` (8 for
   `[5, 4, 3]`), so that no leaf node straddles a chunk boundary.

3. **Node-aligned chunk shape (recommended).** The `z`, `y` and `x` chunk
   extents SHOULD additionally be an integer multiple of, or an integer divisor
   of, the lower internal node extent
   `1 << (tree_config[1] + tree_config[2])` (128 for `[5, 4, 3]`). A chunk of
   exactly one node extent is a subtree with a dense top level, traversable
   without searching the root node's tile table; other shapes satisfying
   requirement 2 are conformant but require a root-level lookup.

   Requirements 2 and 3 constrain only the three spatial dimensions; a
   `channel` dimension is governed by requirement 6 and a `grid` dimension by
   requirement 7. Where a `transpose` precedes this codec the constrained
   extents are `chunk_shape[order[i]]` for the relevant `i`, since `transpose`
   permutes extents without changing them.

4. **Disjoint coverage.** Each of a chunk's grids MUST contain active voxels
   only within that chunk's spatial bounds, so for every index along the `grid`
   dimensions the array's active set is the disjoint union of its chunks'
   active sets.

5. **Uniform configuration.** Every grid of every chunk of an array MUST use
   the same `grid_type`, `tree_config`, `stats`, and background value.

6. **Whole vectors per chunk.** The chunk extent along a `channel` dimension
   MUST equal the array extent along it, so that no chunk holds a partial
   vector.

7. **Grids addressable by index.** A chunk's grid count MUST equal the product
   of its extents along the `grid` dimensions, and grid `n` of the buffer MUST
   correspond to the C-order ravel of the chunk-local `grid` indices, as
   specified in [Grid enumeration](#grid-enumeration).

### Sparsity and losslessness

Decoding produces the grid's active values composited over the array's
`fill_value`: every voxel the grid does not represent decodes as `fill_value`.

- With `lossless: true` (`tolerance: 0`), an encoder MUST leave a voxel
  inactive only if it exactly equals the background; decoding reproduces the
  input bit-for-bit.
- With `lossless: false` and `tolerance: t > 0`, an encoder MAY additionally
  leave inactive any voxel within `t` of the background.
- The quantized grid types are lossy in the value domain independently of
  `tolerance`.

`tolerance` is a topology threshold, not a value threshold: voxels that remain
active are stored at full precision, subject to `grid_type`.

## Interaction with other codecs

[`transpose`](../transpose/README.md) MAY precede this codec. No other
`array -> array` codec may, and implementations SHOULD reject such a chain when
the array metadata is parsed:

- [`reshape`](../reshape/README.md) does not preserve dimension identity, so
  no role describes its output. Composing it with `transpose` does not make it
  conformant.
- Value-domain codecs, [`cast_value`](../cast_value/README.md) and
  [`scale_offset`](../scale_offset/README.md) among them, would leave grid
  values and background disagreeing with `data_type` and `fill_value`.

`bytes -> bytes` codecs compose freely.

## Reader pass-through

A reader MAY retain the buffer rather than decode it to a dense array, and
traverse it directly. Such a reader skips the `array -> array` decode stage, so
it MUST compose the permutation of any preceding
[`transpose`](../transpose/README.md) into its own coordinate mapping.

This is most useful for direct GPU rendering applications, and skipping subtrees by value range requires `stats` of `minmax` or `all`;
encoders targeting such readers SHOULD write one of those rather than `none`.

## Examples

`float32`, leaf- and node-aligned 128³ chunks, lossless, with per-node
min/max:

```json
{
  "shape": [1024, 2048, 2048],
  "data_type": "float32",
  "fill_value": 0.0,
  "chunk_grid": {
    "name": "regular",
    "configuration": { "chunk_shape": [128, 128, 128] }
  },
  "codecs": [
    {
      "name": "nanovdb",
      "configuration": {
        "dimension_roles": ["z", "y", "x"],
        "grid_type": "Float",
        "tree_config": [5, 4, 3],
        "index_space": "global",
        "stats": "minmax",
        "lossless": true,
        "tolerance": 0.0
      }
    },
    { "name": "zstd", "configuration": { "level": 3, "checksum": false } }
  ]
}
```

A segmentation mask, topology only:

```json
{
  "codecs": [
    {
      "name": "nanovdb",
      "configuration": {
        "dimension_roles": ["z", "y", "x"],
        "grid_type": "Mask",
        "tree_config": [5, 4, 3],
        "index_space": "global",
        "stats": "bbox",
        "lossless": true,
        "tolerance": 0.0
      }
    }
  ]
}
```

Quantized and sparsified with a recorded tolerance:

```json
{
  "codecs": [
    {
      "name": "nanovdb",
      "configuration": {
        "dimension_roles": ["z", "y", "x"],
        "grid_type": "Fp16",
        "tree_config": [5, 4, 3],
        "index_space": "global",
        "stats": "all",
        "lossless": false,
        "tolerance": 12.0
      }
    },
    { "name": "zstd", "configuration": { "level": 3, "checksum": false } }
  ]
}
```

A `[Z, Y, X, C]` displacement field with `C = 3` as a `Vec3f` grid; the chunk
covers `channel` in full:

```json
{
  "shape": [512, 1024, 1024, 3],
  "data_type": "float32",
  "fill_value": 0.0,
  "chunk_grid": {
    "name": "regular",
    "configuration": { "chunk_shape": [128, 128, 128, 3] }
  },
  "codecs": [
    {
      "name": "nanovdb",
      "configuration": {
        "dimension_roles": ["z", "y", "x", "channel"],
        "grid_type": "Vec3f",
        "tree_config": [5, 4, 3],
        "index_space": "global",
        "stats": "bbox",
        "lossless": true,
        "tolerance": 0.0
      }
    },
    { "name": "zstd", "configuration": { "level": 3, "checksum": false } }
  ]
}
```

A `[T, Z, Y, X]` series with the leading dimension in the `grid` role: four
grids per chunk buffer.

```json
{
  "shape": [64, 1024, 2048, 2048],
  "data_type": "float32",
  "fill_value": 0.0,
  "dimension_names": ["t", "z", "y", "x"],
  "chunk_grid": {
    "name": "regular",
    "configuration": { "chunk_shape": [4, 128, 128, 128] }
  },
  "codecs": [
    {
      "name": "nanovdb",
      "configuration": {
        "dimension_roles": ["grid", "z", "y", "x"],
        "grid_type": "Float",
        "tree_config": [5, 4, 3],
        "index_space": "global",
        "stats": "minmax",
        "lossless": true,
        "tolerance": 0.0
      }
    },
    { "name": "zstd", "configuration": { "level": 3, "checksum": false } }
  ]
}
```

The same with a three-component value per voxel, `[T, Z, Y, X, C]`:

```json
{
  "shape": [64, 1024, 2048, 2048, 3],
  "data_type": "float32",
  "fill_value": 0.0,
  "dimension_names": ["t", "z", "y", "x", "c"],
  "chunk_grid": {
    "name": "regular",
    "configuration": { "chunk_shape": [4, 128, 128, 128, 3] }
  },
  "codecs": [
    {
      "name": "nanovdb",
      "configuration": {
        "dimension_roles": ["grid", "z", "y", "x", "channel"],
        "grid_type": "Vec3f",
        "tree_config": [5, 4, 3],
        "index_space": "global",
        "stats": "bbox",
        "lossless": true,
        "tolerance": 0.0
      }
    },
    { "name": "zstd", "configuration": { "level": 3, "checksum": false } }
  ]
}
```

An OME-Zarr `[T, C, Z, Y, X]` array with `C = 3`, where `transpose` makes this
codec's input `[t, z, y, x, c]`. `dimension_names` is in the array's order and
`dimension_roles` in the transposed order; `order` relates them:

```json
{
  "shape": [64, 3, 1024, 2048, 2048],
  "data_type": "float32",
  "fill_value": 0.0,
  "dimension_names": ["t", "c", "z", "y", "x"],
  "chunk_grid": {
    "name": "regular",
    "configuration": { "chunk_shape": [4, 3, 128, 128, 128] }
  },
  "codecs": [
    { "name": "transpose", "configuration": { "order": [0, 2, 3, 4, 1] } },
    {
      "name": "nanovdb",
      "configuration": {
        "dimension_roles": ["grid", "z", "y", "x", "channel"],
        "grid_type": "Vec3f",
        "tree_config": [5, 4, 3],
        "index_space": "global",
        "stats": "bbox",
        "lossless": true,
        "tolerance": 0.0
      }
    },
    { "name": "zstd", "configuration": { "level": 3, "checksum": false } }
  ]
}
```

## Example data

TBD. Fixtures are planned alongside the reference implementation: a small
`float32` volume and a `Mask` volume, each with a `fixtures.json` recording the
expected decoded values and active/inactive topology.

## Interoperability and compatibility

- The buffer is a plain NanoVDB grid, readable by NanoVDB independently of Zarr
  once the `bytes -> bytes` stage has been undone. With a compressor in the
  chain a stored chunk is not a NanoVDB buffer as it sits.
- Grids can be produced from OpenVDB grids with `nanovdb::createNanoGrid`, and
  from dense arrays with NanoVDB's build tools. OpenVDB's `.vdb` serialization
  is not interchangeable with a NanoVDB buffer and must be converted.
- A buffer carries a version and declares its grid type, so a reader can reject
  a buffer written by a newer NanoVDB or holding a value type it does not
  handle. It does not declare its tree configuration; see
  [`tree_config`](#tree_config).
- Reference implementation: in progress, targeting a Zarr reader that passes
  the buffer through to a GPU without densifying it.

## Change log

No changes yet.

## Current maintainers

- TBD
