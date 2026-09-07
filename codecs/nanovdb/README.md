# nanovdb codec

Defines an `array -> bytes` codec that encodes an array chunk as a
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

Because a NanoVDB buffer is self-describing, decoders MUST derive grid
geometry, grid type, and tree configuration from the buffer itself; MUST verify
that they are consistent with the array metadata and this configuration; and
MUST return an error on mismatch.

### `grid_type`

The NanoVDB `GridType` of the encoded grid: one of `Float`, `Double`, `Int16`,
`Int32`, `Int64`, `UInt32`, `Mask`, `Fp4`, `Fp8`, `Fp16`, `FpN`, `Vec3f`,
`Vec3d`, `Vec4f`, `Vec4d`. Not redundant with the array's `data_type`: the
quantized types (`Fp4`, `Fp8`, `Fp16`, `FpN`) encode a `float32` array in fewer
bits per voxel, and the vector types pair a `float32` or `float64` array with a
component dimension. See [Supported data types](#supported-data-types).

### `tree_config`

An array of the tree's node branching factors, as the `Log2Dim` of each level
from the root side to the leaf. MUST be `[5, 4, 3]`, giving 8³ leaves, 128³
lower internal nodes and 4096³ upper internal nodes. Every extent the
[grid coherence](#grid-coherence-across-chunks) requirements refer to derives
from this array: the leaf extent is `1 << tree_config[2]` and the lower
internal node extent is `1 << (tree_config[1] + tree_config[2])`.

`[5, 4, 3]` is the only registered configuration, for two reasons beyond its
being OpenVDB's default:

- It is the only one the official released NanoVDB implementation reads or
  writes.
- A reader cannot recover the configuration from the buffer.

The field is therefore declared explicitly rather than assumed, so that a
reader can reject what it does not implement instead of misreading it, and so
that another configuration can be registered later without ambiguity. Any
future value MUST also have exactly three entries: NanoVDB's tree depth is not
parameterized, and the [grid coherence](#grid-coherence-across-chunks)
requirements assume the three levels above.

### `index_space`

One of `global` or `chunk_local`: whether each chunk's grid uses the array's
global index coordinates or is rebased to the chunk's own origin. `global` is
REQUIRED for [grid coherence](#grid-coherence-across-chunks) and SHOULD be
used.

### `stats`

One of `none`, `bbox`, `minmax`, `all`: the per-node statistics the grid
carries. `bbox` includes active bounding boxes; `minmax` adds per-node minimum
and maximum values; `all` adds average and standard deviation. Empty-space
skipping and range estimation depend on `minmax` or `all`; see
[Reader pass-through](#reader-pass-through-non-normative).

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

The chunk MUST have exactly three spatial dimensions, optionally followed by a
single component dimension for vector-valued grids; rank alone disambiguates
the two forms. Spatial dimensions map to NanoVDB coordinates in C order, the
innermost spatial dimension being the NanoVDB `x` axis:

| chunk shape       | grid value | NanoVDB coordinate of array index      |
| ----------------- | ---------- | -------------------------------------- |
| `[nz, ny, nx]`    | scalar     | `[k, j, i]` → `(i, j, k)`              |
| `[nz, ny, nx, c]` | `c`-vector | `[k, j, i, c]` → `(i, j, k)` comp. `c` |

The component dimension MUST be the innermost (unit-stride) dimension, and its
extent `c` MUST be `3` or `4` and match the vector `grid_type`. NanoVDB stores
a voxel's vector components contiguously, so this layout makes the array's
memory layout and the grid's identical.

Encoders and decoders MUST return an error if the chunk is not one of these two
forms, if `c` disagrees with `grid_type`, or if the grid's index bounding box
is inconsistent with the chunk's position and spatial shape.

Because no `array -> array` codec may precede this one (see
[Interaction with other codecs](#interaction-with-other-codecs)), the array
must natively be laid out in one of these forms. Arrays carrying a time axis,
or more channels than a vector grid can hold, SHOULD place those on separate
arrays so that each spatial volume is encoded as its own coherent grid.

These rules apply to the inner chunk shape when this codec is used as the
array-to-bytes codec within the
[`sharding_indexed`](../sharding_indexed/README.md) codec.

## Supported data types

Scalar grids take a rank-3 chunk:

| Zarr `data_type` | `grid_type`                          | Notes                                         |
| ---------------- | ------------------------------------ | --------------------------------------------- |
| `float32`        | `Float`, `Fp4`, `Fp8`, `Fp16`, `FpN` | Quantized types are lossy                     |
| `float64`        | `Double`                             |                                               |
| `int16`          | `Int16`                              |                                               |
| `int32`          | `Int32`                              |                                               |
| `int64`          | `Int64`                              |                                               |
| `uint32`         | `UInt32`                             |                                               |
| `bool`           | `Mask`                               | Topology only; the value _is_ the active mask |

Vector grids take a rank-4 chunk whose innermost dimension is the component
dimension, with extent `c`:

| Zarr `data_type` | `c` | `grid_type` |
| ---------------- | --- | ----------- |
| `float32`        | `3` | `Vec3f`     |
| `float64`        | `3` | `Vec3d`     |
| `float32`        | `4` | `Vec4f`     |
| `float64`        | `4` | `Vec4d`     |

Data types not listed above are supported only through promotion:
implementations MAY promote a narrower data type to a wider `grid_type` (for
example `uint8` or `int8` to `Int16`), but MUST record the `grid_type` actually
written and MUST NOT promote across the scalar/vector boundary. Zarr data types
with no NanoVDB counterpart (complex and variable-length types among them) are
not supported.

Two NanoVDB grid families are out of scope: matrix grids, which would require
two component dimensions (`[Z, Y, X, 3, 3]`) and therefore a revision of the
[chunk shape rules](#supported-chunk-shapes); and integer vector grids
(`Vec3i` and similar), a plausible future addition pending confirmation of
which are available across the NanoVDB versions this codec targets.

For vector grids, two consequences of Zarr's `fill_value` being a scalar:

- The grid's background value MUST be the vector all of whose components equal
  `fill_value`; a background with unequal components cannot be expressed in the
  array metadata and MUST NOT be written.
- A voxel is active or inactive as a whole. An encoder MUST treat a voxel as
  non-background if _any_ component differs from `fill_value` by more than
  `tolerance`.

> [!NOTE]
> **Open question for review.** `stats: minmax` is not obviously well defined
> for vector grids, which have no natural total order. Componentwise minima and
> maxima are the plausible reading, but this proposal does not yet fix it; the
> conservative option is to restrict vector grids to `stats: none` or `bbox`.

## Format and algorithm

### Encoded representation

The encoded chunk is a single NanoVDB grid buffer: the in-memory layout
produced by NanoVDB's grid builders, written verbatim.

- The buffer MUST contain exactly one grid, MUST be little-endian, and MUST
  begin with a valid NanoVDB grid magic number and version header. The magic
  numbers, version encoding, and structure layouts are defined normatively by
  [`nanovdb/NanoVDB.h`](https://github.com/AcademySoftwareFoundation/openvdb/blob/master/nanovdb/nanovdb/NanoVDB.h);
  this document does not restate them.
- The buffer MUST NOT be wrapped in the `nanovdb::io` file container, whose
  file headers and optional compression duplicate the Zarr chunk key and
  `bytes -> bytes` codecs respectively.
- The grid's background value MUST equal the array's `fill_value`, so that
  inactive voxels decode identically across every chunk.

### Grid coherence across chunks

Encoding each chunk as an unrelated grid — its own origin, alignment, and tree
shape — would leave readers unable to treat the chunks as parts of one
hierarchy. The following requirements prevent that. An array using this codec
MUST satisfy all of them, and each is checkable from the array metadata and
chunk buffers alone.

1. **One index space.** Each chunk's grid MUST express voxel coordinates in the
   array's global index space (array index `[k, j, i]` occupies NanoVDB
   coordinate `(i, j, k)` regardless of which chunk holds it); grids MUST NOT
   be rebased to a chunk-local origin. Chunks of a region then compose by
   union, and a writer can produce them by splitting one global grid.

2. **Leaf-aligned chunk grid.** Every spatial chunk dimension, and the chunk
   grid origin in each spatial dimension, MUST be an integer multiple of the
   leaf extent `1 << tree_config[2]` (8 for `[5, 4, 3]`), so that no leaf node
   straddles a chunk boundary.

3. **Node-aligned chunk shape (recommended).** Spatial chunk dimensions SHOULD
   additionally be an integer multiple of, or an integer divisor of, the lower
   internal node extent `1 << (tree_config[1] + tree_config[2])` (128 for
   `[5, 4, 3]`). A chunk that is an exact node extent
   is a clean subtree with a dense top level, addressable without searching the
   root node's tile table; other shapes satisfying requirement 2 remain
   conformant at the cost of a root-level lookup per traversal. The component
   dimension of a vector-valued array is exempt from requirements 2 and 3 and
   is governed by requirement 6.

4. **Disjoint coverage.** A chunk's grid MUST contain active voxels only within
   that chunk's bounds, so the array's active set is the disjoint union of its
   chunks' active sets.

5. **Uniform configuration.** Every chunk of an array MUST use the same
   `grid_type`, `tree_config`, `stats`, and background value, letting a reader
   specialize its traversal (including compiling a shader) once per array.

6. **Whole vectors per chunk.** For a vector-valued array, the chunk extent
   along the component dimension MUST equal the array extent along it, so that
   no chunk holds a partial vector.

Requirement 3 is a recommendation because its tradeoff is reader-side cost
rather than correctness; the rest are what allow the chunks of an array to be
treated as one logical grid without inspecting every chunk.

### Sparsity and losslessness

Decoding produces the grid's active values composited over the array's
`fill_value`: every voxel the grid does not represent decodes as `fill_value`.

- With `lossless: true` (`tolerance: 0`), an encoder MUST leave a voxel
  inactive only if it exactly equals the background; decoding reproduces the
  input array bit-for-bit.
- With `lossless: false` and `tolerance: t > 0`, an encoder MAY additionally
  leave inactive any voxel within `t` of the background — the useful case for
  noisy data, recorded in the metadata rather than left as an encoder artifact.
- The quantized grid types are lossy in the value domain independently of
  `tolerance`.

`tolerance` is a topology threshold, not a value threshold: voxels that remain
active are stored at full precision (subject to `grid_type`).

## Interaction with other codecs

No `array -> array` codec may precede this codec, and implementations SHOULD
reject such a chain when the array metadata is parsed:

1. Codecs that permute or reshape dimensions
   ([`transpose`](../transpose/README.md), [`reshape`](../reshape/README.md))
   break requirement 1 of [grid coherence](#grid-coherence-across-chunks),
   which ties NanoVDB coordinates to array index coordinates.
2. `array -> array` codecs decode _after_ the `array -> bytes` codec. A reader
   taking the pass-through path below never materializes a dense array for them
   to operate on, so permitting them would forfeit that optimization for
   exactly the arrays this codec serves. This excludes even value-domain codecs
   such as [`cast_value`](../cast_value/README.md) and
   [`scale_offset`](../scale_offset/README.md); their effect belongs in how the
   array was written.

`bytes -> bytes` codecs compose freely and are the right place for general
compression and checksums.

## Reader pass-through (non-normative)

Because the encoded buffer is already a pointer-free spatial index, a reader
MAY keep it rather than decode it to a dense array — including handing it to a
GPU as-is and traversing it in a shader. The coherence requirements make this
safe without inspecting the whole array: traversal is specialized once per
array (requirement 5), chunks are addressed without coordinate arithmetic (1),
no leaf spans a chunk boundary (2), and the root-level search can be skipped
when requirement 3 holds. Such readers benefit from `stats` being `minmax` or
`all`, which enables skipping subtrees outside the value range of interest;
encoders targeting them SHOULD write those rather than `none`.

## Examples

A `float32` volume in leaf- and node-aligned 128³ chunks, losslessly sparsified
against a background of `0.0`, with per-node min/max statistics:

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

A segmentation mask, storing topology only:

```json
{
  "codecs": [
    {
      "name": "nanovdb",
      "configuration": {
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

Thresholded fluorescence, quantized and sparsified with a recorded tolerance:

```json
{
  "codecs": [
    {
      "name": "nanovdb",
      "configuration": {
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

A sparse `float32` displacement field shaped `[Z, Y, X, C]` with `C = 3`,
encoded as a `Vec3f` grid; the component dimension is innermost and covered in
full by the chunk shape, so every chunk holds whole vectors:

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

- The encoded buffer is a plain NanoVDB grid, readable by NanoVDB and any tool
  built on it, independently of Zarr.
- Grids can be produced from OpenVDB grids with `nanovdb::createNanoGrid`, and
  from dense arrays with NanoVDB's build tools. OpenVDB's own `.vdb`
  serialization is _not_ interchangeable with a NanoVDB buffer and must be
  converted.
- NanoVDB buffers are versioned and self-describing, so a reader can detect a
  buffer written by a newer NanoVDB than it supports and fail cleanly.
- Reference implementation: in progress, targeting a Zarr reader that passes
  the buffer through to a GPU without densifying it.

## Change log

No changes yet.

## Current maintainers

- TBD
