# nanovdb codec

Defines an `array -> bytes` codec that encodes an array chunk as a
[NanoVDB](https://www.openvdb.org/documentation/doxygen/NanoVDB_MainPage.html)
grid buffer.

NanoVDB is the linearized, pointer-free serialization of an
[OpenVDB](https://www.openvdb.org/) tree: a fixed-depth spatial hierarchy in
which regions holding a single repeated value are stored once as a tile rather
than as voxels. For volumes that are mostly background — segmentation masks,
level sets, sparse labels, distance fields, thresholded fluorescence, sparse
vector fields such as velocity or displacement — this is both a large size
reduction and a spatial index that a reader can traverse directly.

This codec is intentionally minimal. It fixes the relationship between an array
chunk and a NanoVDB grid, and — critically — constrains that relationship so
that the grids of every chunk in an array are **mutually coherent**: they share
one index space and one node lattice, so the chunks tile a single logical VDB
rather than being an unrelated collection of small ones. See
[Grid coherence across chunks](#grid-coherence-across-chunks).

> This document is a proposed extension. It is licensed under the
> [Creative Commons Attribution 3.0 Unported License](https://creativecommons.org/licenses/by/3.0/).

> [!NOTE]
> **Status: draft.** This is an early proposal opened to solicit feedback. The
> reference implementation is in progress; see
> [Interoperability and compatibility](#interoperability-and-compatibility).

## Codec name

The value of the `name` member in the codec object MUST be `nanovdb`.

## Configuration parameters

The `configuration` object is REQUIRED and records the encoding parameters used
to produce the buffer.

The configuration is a **closed set**: implementations MUST NOT emit members
other than those below, and a store is non-conformant if `configuration`
contains unrecognized members. Encoders SHOULD write every member explicitly
rather than relying on defaults, so the metadata is the definitive record of how
chunks were encoded.

A NanoVDB buffer is self-describing, so decoders MUST derive grid geometry, grid
type, and tree configuration from the buffer itself and MUST NOT rely on these
members to parse it. Decoders MUST, however, verify that what the buffer
declares is consistent with both the array metadata and this configuration, and
MUST return an error on mismatch.

**Always present**:

- **`grid_type`** (string): the NanoVDB `GridType` of the encoded grid, as one
  of `Float`, `Double`, `Int16`, `Int32`, `Int64`, `UInt32`, `Mask`, `Fp4`,
  `Fp8`, `Fp16`, `FpN`, `Vec3f`, `Vec3d`, `Vec4f`, `Vec4d`. This is not
  redundant with the array's `data_type`: the quantized types (`Fp4`, `Fp8`,
  `Fp16`, `FpN`) encode a `float32` array in fewer bits per voxel, and the
  vector types pair a `float32` or `float64` array with a component dimension.
  See [Supported data types](#supported-data-types).
- **`tree_config`** (string): the node branching factors of the tree, written as
  the `Log2Dim` values from root side to leaf, hyphen-separated. MUST be
  `5-4-3` (OpenVDB's default: 8³ leaves, 128³ lower internal nodes, 4096³ upper
  internal nodes). Declared explicitly so that other configurations can be
  registered later without ambiguity.
- **`index_space`** (string, one of `global`, `chunk_local`): whether each
  chunk's grid is expressed in the array's global index coordinates or rebased
  to the chunk's own origin. `global` is REQUIRED for coherence and SHOULD be
  used; see [Grid coherence across chunks](#grid-coherence-across-chunks).
- **`stats`** (string, one of `none`, `bbox`, `minmax`, `all`): which per-node
  statistics the grid carries. `bbox` includes active bounding boxes; `minmax`
  adds per-node minimum and maximum values; `all` adds average and standard
  deviation. Readers that perform empty-space skipping or range estimation
  depend on `minmax` or `all`; see
  [Reader pass-through](#reader-pass-through-non-normative).
- **`lossless`** (boolean): whether decoding reproduces the encoded array
  exactly. When `true`, `tolerance` MUST be `0` and `grid_type` MUST NOT be one
  of the quantized types. When `false`, `tolerance` MUST be greater than `0` or
  `grid_type` MUST be a quantized type.
- **`tolerance`** (number, ≥ 0): the maximum absolute deviation from the
  background value at which the encoder may leave a voxel inactive. `0` means
  only voxels exactly equal to the background may be dropped. See
  [Sparsity and losslessness](#sparsity-and-losslessness).

## Encoded representation

The encoded chunk is a single NanoVDB grid buffer: the in-memory layout produced
by NanoVDB's grid builders, written verbatim.

- The buffer MUST contain exactly **one** grid.
- The buffer MUST be little-endian.
- The buffer MUST begin with a valid NanoVDB grid magic number and version
  header. The magic numbers, version encoding, and structure layouts are defined
  normatively by
  [`nanovdb/NanoVDB.h`](https://github.com/AcademySoftwareFoundation/openvdb/blob/master/nanovdb/nanovdb/NanoVDB.h)
  in the OpenVDB repository; this document does not restate them.
- The buffer MUST NOT be wrapped in the `nanovdb::io` file container. That
  container adds file-level headers and its own optional compression, both of
  which duplicate responsibilities that belong to the Zarr chunk key and to
  `bytes -> bytes` codecs respectively.
- The grid's **background value** MUST equal the array's `fill_value`. This is
  what makes inactive voxels decode correctly, and makes the meaning of
  "inactive" identical across every chunk of the array.


## Array shape contract

The chunk MUST have exactly three spatial dimensions, optionally followed by a
single **component dimension** for vector-valued grids. Spatial dimensions map
to NanoVDB coordinates in C order, so that the innermost spatial dimension is
the NanoVDB `x` axis:

| chunk shape       | grid value  | NanoVDB coordinate of array index      |
| ----------------- | ----------- | -------------------------------------- |
| `[nz, ny, nx]`    | scalar      | `[k, j, i]` → `(i, j, k)`              |
| `[nz, ny, nx, c]` | `c`-vector  | `[k, j, i, c]` → `(i, j, k)` comp. `c` |

The two forms are disambiguated by rank alone: a rank-3 chunk is scalar-valued,
a rank-4 chunk is vector-valued. The component extent `c` MUST be `3` or `4`,
matching the vector `grid_type` (see
[Supported data types](#supported-data-types)).

The component dimension MUST be the **innermost** (unit-stride) dimension. This
is not an arbitrary choice: NanoVDB stores a vector-valued leaf as an array of
vectors, with a voxel's components contiguous, so an innermost component
dimension makes the array's memory layout and the grid's identical.

Encoders and decoders MUST return an error if the chunk is not one of these two
forms, if `c` is neither `3` nor `4`, if `c` disagrees with the component count
implied by `grid_type`, or if the grid's index bounding box is inconsistent with
the chunk's position and spatial shape.

Because no `array -> array` codec may precede this one (see
[Interaction with other codecs](#interaction-with-other-codecs)), the array must
already be laid out in one of these two forms — the layout cannot be produced by
a preceding `transpose` or `reshape`. Arrays carrying a time axis, or more
channels than a vector grid can hold, SHOULD place those on separate arrays so
that each spatial volume is encoded as its own coherent grid.

These rules apply to the inner chunk shape when this codec is used as the
array-to-bytes codec within the [`sharding_indexed`](../sharding_indexed/README.md)
codec.

## Grid coherence across chunks

A NanoVDB grid is normally a *global* structure with its own index space and node
lattice. Splitting a volume into Zarr chunks and encoding each independently
would ordinarily destroy that: each chunk would carry an unrelated grid with its
own origin, its own alignment, and potentially its own tree shape. Readers would
then have to treat each chunk as an opaque volume rather than as a piece of one
hierarchy, and no per-chunk grid could be traversed without first resolving
where it sits.

The following requirements remove that problem at the format level. An array
using this codec MUST satisfy all of them, and a validator can check every one
of them from the array metadata plus the chunk buffers alone.

1. **One index space.** When `index_space` is `global`, each chunk's grid MUST
   express voxel coordinates in the array's global index space — that is, a
   voxel at array index `[k, j, i]` occupies NanoVDB coordinate `(i, j, k)`,
   regardless of which chunk holds it. Grids MUST NOT be rebased to a
   chunk-local origin. A reader can therefore compose the chunks of a region by
   union, with no coordinate arithmetic, and a writer can produce them by
   splitting one global grid.

2. **Leaf-aligned chunk grid.** Every **spatial** chunk dimension MUST be an
   integer multiple of the leaf extent implied by `tree_config` (8 for `5-4-3`),
   and the array's chunk grid origin MUST be a multiple of that extent in each
   spatial dimension. No leaf node may straddle a chunk boundary, so every leaf
   belongs to exactly one chunk.

3. **Node-aligned chunk shape (recommended).** Spatial chunk dimensions SHOULD
   additionally be an integer multiple of, or an integer divisor of, the lower
   internal node extent (128 for `5-4-3`). A chunk that is an exact node extent
   is a clean subtree with a **dense** top level, which lets a reader address
   the top level by direct indexing instead of searching the root node's tile
   table. Chunk shapes that satisfy requirement 2 but not this one remain
   conformant, at the cost of a root-level lookup per traversal.

   The component dimension of a vector-valued array is exempt from requirements
   2 and 3, and is instead governed by requirement 6.

4. **Disjoint coverage.** A chunk's grid MUST contain active voxels only within
   that chunk's bounds. Combined with requirement 1, the active set of the array
   is exactly the disjoint union of the active sets of its chunks, and no voxel
   is represented twice.

5. **Uniform configuration.** Every chunk of an array MUST use the same
   `grid_type`, `tree_config`, `stats`, and background value. A reader may
   therefore specialize its traversal — including compiling a shader — once per
   array rather than once per chunk.

6. **Whole vectors per chunk.** For a vector-valued array, the chunk extent
   along the component dimension MUST equal the array extent along it, so that
   no chunk holds a partial vector. A voxel's components are therefore always
   resolvable from a single chunk, and a grid never has to be assembled across
   chunk boundaries to read one value.

Requirement 3 is the only one stated as a recommendation, because the tradeoff is
a reader-side cost rather than a correctness problem. The remaining five are
what allow the chunks of an array to be treated as one logical grid, and a
reader MAY rely on them without inspecting every chunk.

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
| `bool`           | `Mask`                               | Topology only; the value *is* the active mask |

Vector grids take a rank-4 chunk whose innermost dimension is the component
dimension, with extent `c` as given below:

| Zarr `data_type` | `c` | `grid_type` |
| ---------------- | --- | ----------- |
| `float32`        | `3` | `Vec3f`     |
| `float64`        | `3` | `Vec3d`     |
| `float32`        | `4` | `Vec4f`     |
| `float64`        | `4` | `Vec4d`     |

This is the natural mapping for an array shaped `[Z, Y, X, C]` with `C = 3` or
`C = 4`: the component dimension is already innermost, so the array's layout and
the grid's are identical and no rearrangement is needed. See
[Array shape contract](#array-shape-contract).

Two consequences of Zarr's `fill_value` being a scalar:

- The grid's background value MUST be the vector all of whose components equal
  `fill_value`. A vector background with unequal components cannot be expressed
  in the array metadata and MUST NOT be written.
- A voxel is active or inactive as a whole; individual components cannot be. An
  encoder MUST treat a voxel as non-background if *any* component differs from
  `fill_value` by more than `tolerance`.

> [!NOTE]
> **Open question for review.** For vector grid types, the semantics of
> `stats: minmax` are not obviously well defined, since vectors have no natural
> total order. Componentwise minima and maxima are the plausible reading, but
> this proposal does not yet fix it, and readers that skip subtrees by value
> range should not assume a scalar ordering. Feedback welcome; the conservative
> option is to restrict vector grids to `stats: none` or `stats: bbox`.

Other data types are not supported. NanoVDB's integer vector and matrix grid
types are candidates for a future revision, pending confirmation of which are
available across the NanoVDB versions this codec targets.

Implementations MAY support narrower types through promotion (for example
`uint8` or `int8` to `Int16`), but MUST record the `grid_type` actually written.

## Sparsity and losslessness

Decoding produces the grid's active values composited over the array's
`fill_value`: every voxel the grid does not represent decodes as `fill_value`.

Whether that round-trips exactly depends on the encoder:

- With `lossless: true` and `tolerance: 0`, an encoder MUST leave a voxel
  inactive only if its value is exactly equal to the background. Decoding then
  reproduces the input array bit-for-bit.
- With `lossless: false` and `tolerance: t > 0`, an encoder MAY additionally
  leave inactive any voxel within `t` of the background. This is the case that
  makes the codec useful on noisy data, where a true background is rare but a
  near-background is common — and it is a deliberate, recorded choice rather
  than an artifact of the encoder.
- The quantized grid types are lossy independently of `tolerance`, in the value
  domain rather than the topology domain.

Note that `tolerance` is a *topology* threshold, not a value threshold: voxels
that remain active are stored at full precision (subject to `grid_type`).
Consumers that need a defensible relationship between the stored volume and the
original measurement should record how `tolerance` was chosen alongside the
array.

## Interaction with other codecs

**No `array -> array` codec may precede this codec.** There are two independent
reasons, and the second applies even to codecs that leave the index space
untouched:

1. A codec that permutes or reshapes dimensions — [`transpose`](../transpose/README.md)
   or [`reshape`](../reshape/README.md) — breaks requirement 1 of
   [Grid coherence](#grid-coherence-across-chunks), which fixes NanoVDB
   coordinates to array index coordinates.
2. `array -> array` codecs are applied in reverse on read, *after* the
   `array -> bytes` codec has been decoded. A reader that takes the pass-through
   path described below never materializes a dense array, so there is nothing
   for such a codec to operate on. Permitting them would make the pass-through
   optimization unavailable for the very arrays this codec exists to serve.

Implementations SHOULD reject such a codec chain when the array metadata is
parsed, rather than at decode time. The practical consequence is that the array
must natively be laid out as `[Z, Y, X]` or `[Z, Y, X, C]`; this codec
deliberately does not offer a way to rearrange it.

Value-domain codecs such as [`cast_value`](../cast_value/README.md) and
[`scale_offset`](../scale_offset/README.md) are `array -> array` codecs and are
therefore also excluded, notwithstanding that they preserve the index space.
Where their effect is wanted, it belongs in how the array was written — noting
in any case that they change what the background value is, and that requirement
5 applies to the background as this codec sees it.

`bytes -> bytes` codecs compose freely and are the right place for general
compression and checksums.

## Reader pass-through (non-normative)

A conventional Zarr reader will decode a chunk to a dense array, which discards
the size advantage as soon as the chunk is in memory. Because the encoded buffer
is already a spatial index, a reader MAY instead keep the buffer and resolve
voxels by traversing it — the buffer is pointer-free and position-independent,
so it can be handed to a GPU as-is and traversed in a shader.

The coherence requirements above are what make this safe to do without
inspecting the whole array: a reader can specialize its traversal once
(requirement 5), address chunks by index without coordinate arithmetic
(requirement 1), assume no leaf spans a boundary (requirement 2), and skip the
root-level search entirely when requirement 3 holds.

Readers doing this benefit from `stats` being `minmax` or `all`, which lets them
skip whole subtrees whose value range falls outside the range currently of
interest, and estimate a display range without sampling. Encoders targeting such
readers SHOULD therefore write `minmax` or `all` rather than `none`.

## Examples

A `float32` volume stored in leaf- and node-aligned 128³ chunks, losslessly
sparsified against a background of `0.0`, with per-node minimum and maximum
statistics:

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
        "tree_config": "5-4-3",
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

A segmentation mask, where only topology is stored:

```json
{
  "codecs": [
    {
      "name": "nanovdb",
      "configuration": {
        "grid_type": "Mask",
        "tree_config": "5-4-3",
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
        "tree_config": "5-4-3",
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
encoded as a `Vec3f` grid. Note that the component dimension is innermost and
that the chunk shape covers it in full, so every chunk holds whole vectors:

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
        "tree_config": "5-4-3",
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
expected decoded values and the active/inactive topology.

## Interoperability and compatibility

- The encoded buffer is a plain NanoVDB grid, so it can be read by NanoVDB
  itself and by any tool built on it, independently of Zarr.
- Grids can be produced from OpenVDB grids with `nanovdb::createNanoGrid`, and
  from dense arrays with NanoVDB's build tools. Note that OpenVDB's own `.vdb`
  serialization is *not* interchangeable with a NanoVDB buffer: it is a
  node-graph format with its own per-node compression, and must be converted.
- Because NanoVDB buffers are versioned and self-describing, a reader can detect
  a buffer written by a newer NanoVDB than it supports and fail cleanly.
- Reference implementation: in progress, targeting a Zarr reader that passes the
  buffer through to a GPU without densifying it. Not yet available; this
  proposal is opened to solicit feedback on the coherence requirements before
  the implementation is finalized.

## Change log

No changes yet.

## Current maintainers

- TBD
