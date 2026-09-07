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

A NanoVDB buffer declares its own geometry and grid type, so decoders MUST take
those from the buffer rather than from this configuration; MUST verify that they
are consistent with the array metadata and with the members below; and MUST
return an error on mismatch.

[`tree_config`](#tree_config) is the exception, because the buffer does not
record it. A decoder MUST reject a declared `tree_config` it does not implement,
and SHOULD check that the buffer's node byte spans agree with the declared
branching factors wherever a level is non-empty.

### `dimension_roles`

An array of role tokens, one per dimension of **this codec's input**, outermost
first. The input is the array chunk after any preceding `array -> array` codec
has been applied, so when a [`transpose`](../transpose/README.md) precedes this
codec the roles are listed in the transposed order, not the array's.
Every dimension MUST be given a role explicitly. The codec never infers roles
from the input's rank, from `dimension_names`, or from any convention layered
above Zarr.

| role          | how this codec encodes the dimension                                                           |
| ------------- | ---------------------------------------------------------------------------------------------- |
| `grid`        | Each index becomes a separate NanoVDB grid within the chunk's buffer.                          |
| `z`, `y`, `x` | A NanoVDB coordinate axis. These three dimensions are the tree's index space.                  |
| `channel`     | Becomes the grid's value type: extent `1` a scalar grid, `3` a `Vec3` grid, `4` a `Vec4` grid. |

The roles MUST appear in this order, and encoders and decoders MUST reject any
other arrangement:

```
[grid … grid]   z   y   x   [channel]
```

That is: zero or more `grid` dimensions, outermost; then exactly one each of
`z`, `y` and `x`, in that order; then at most one `channel` dimension, which
MUST be the innermost (unit-stride) dimension. A `channel` dimension MUST have
extent `1`, `3` or `4`, matching `grid_type` per
[Supported data types](#supported-data-types); an array with no `channel`
dimension is scalar, exactly as one with a `channel` dimension of extent `1`.
The strict order above constrains this codec's input, not the array. An array
whose dimensions are in any other order reaches it through a preceding
[`transpose`](../transpose/README.md), which is permitted for exactly this
purpose; see
[Interaction with other codecs](#interaction-with-other-codecs). Since
`transpose` defines its output dimension `i` to be its input dimension
`order[i]`, the two line up as `dimension_roles[i]` ↔ array dimension
`order[i]` — so `order` follows from the roles the array's dimensions are meant
to carry, and an implementation MAY derive it. OME-Zarr `[t, c, z, y, x]` with
`c = 3` gives `order: [0, 2, 3, 4, 1]`; an `[x, y, z]` array gives
`order: [2, 1, 0]`.

Note that with a `transpose` in the chain `dimension_roles` is **not**
index-aligned with `shape`, `chunk_shape` or `dimension_names`, which are in
the array's own order; `order` is the only thing relating them.

### `grid_type`

The NanoVDB `GridType` of the encoded grid: one of `Float`, `Double`, `Int16`,
`Int32`, `Int64`, `UInt32`, `Mask`, `Fp4`, `Fp8`, `Fp16`, `FpN`, `Vec3f`,
`Vec3d`, `Vec4f`, `Vec4d`. Not redundant with the array's `data_type`: the
quantized types (`Fp4`, `Fp8`, `Fp16`, `FpN`) encode a `float32` array in fewer
bits per voxel, and the vector types pair a `float32` or `float64` array with a
`channel` dimension. See [Supported data types](#supported-data-types).

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
- A reader cannot recover the configuration from the buffer. NanoVDB records
  branching factors nowhere: not in the grid header, and not in the
  `nanovdb::io` file header either, whose per-grid metadata carries node and
  tile _counts_ rather than node dimensions. Counts do not determine the
  `Log2Dim` values, and the arithmetic that would recover node sizes from them
  breaks down for a level with no nodes — the sparse case this codec serves.
  Wrapping the buffer in the file container is therefore not a way to make it
  self-describing.

The field is therefore declared explicitly rather than assumed, so that a
reader can reject what it does not implement instead of misreading it, and so
that another configuration can be registered later without ambiguity. Note
that declaring it buys a clean rejection, not compatibility: because a NanoVDB
reader's tree type is fixed when it is compiled, no amount of metadata makes
another configuration readable by an implementation built for this one. Keeping
the declaration in the array metadata rather than the buffer lets a decoder
reject an array before fetching any chunk.

Any future value MUST also have exactly three entries: NanoVDB's tree depth is
not parameterized, and the
[grid coherence](#grid-coherence-across-chunks) requirements assume the three
levels above.

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
[Reader pass-through](#reader-pass-through).

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
[`dimension_roles`](#dimension_roles), and each dimension is constrained by its
role:

| role      | chunk extent                                                                                                                                |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| `grid`    | Any extent. The chunk's buffer holds one grid per index, so the grid count is the product of the chunk extents along all `grid` dimensions. |
| `z y x`   | Constrained by requirements 2 and 3 of [grid coherence](#grid-coherence-across-chunks): leaf-aligned, and node-aligned where practical.     |
| `channel` | MUST equal the array extent along that dimension, so that no chunk holds a partial vector (requirement 6).                                  |

The three spatial dimensions map to NanoVDB coordinates with the innermost
being the `x` axis. Writing `t` for the index along a single `grid` dimension
and `c` for the `channel` index:

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

Scalar grids take an array with no `channel` dimension, or a `channel`
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

A `channel` extent of `2`, or greater than `4`, has no NanoVDB counterpart and
MUST be rejected. An axis of that width whose values belong to one voxel has no
representation in this codec; an axis whose indices are independent fields
SHOULD be given the `grid` role instead, which places each on its own grid.

Data types not listed above are supported only through promotion:
implementations MAY promote a narrower data type to a wider `grid_type` (for
example `uint8` or `int8` to `Int16`), but MUST record the `grid_type` actually
written and MUST NOT promote across the scalar/vector boundary. Zarr data types
with no NanoVDB counterpart (complex and variable-length types among them) are
not supported.

Two NanoVDB grid families are out of scope: matrix grids, which would require
two `channel` dimensions (`[Z, Y, X, 3, 3]`) and therefore a revision of
[`dimension_roles`](#dimension_roles), which permits at most one; and integer
vector grids
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

The encoded chunk is a NanoVDB grid buffer: the in-memory layout produced by
NanoVDB's grid builders, written verbatim.

- The buffer MUST be little-endian and MUST begin with a valid NanoVDB grid
  magic number and version header. The magic numbers, version encoding, and
  structure layouts are defined normatively by
  [`nanovdb/NanoVDB.h`](https://github.com/AcademySoftwareFoundation/openvdb/blob/master/nanovdb/nanovdb/NanoVDB.h);
  this document does not restate them.
- The buffer MUST NOT be wrapped in the `nanovdb::io` file container, whose
  file headers and optional compression duplicate the Zarr chunk key and
  `bytes -> bytes` codecs respectively. Nothing is given up by omitting it: a
  raw grid buffer identifies itself with its own magic number, distinct from
  the container's, and NanoVDB reads raw buffers directly — including
  addressing one grid of a multi-grid buffer. The container's remaining fields
  are a name lookup key and geometry that is recoverable from the grid itself.
  It records no tree configuration either; see
  [`tree_config`](#tree_config).
- Every grid's background value MUST equal the array's `fill_value`, so that
  inactive voxels decode identically across every chunk.

#### Grid enumeration

An array whose [`dimension_roles`](#dimension_roles) contain no `grid`
dimension encodes one grid per chunk. Otherwise the chunk's buffer holds
`mGridCount` grids, laid out exactly as `nanovdb::mergeGrids` produces them:

- The grid count MUST equal the product of the chunk's extents along its
  `grid` dimensions, and MUST be recorded as `mGridCount` in **every** grid's
  header, not only the first.
- Grids MUST be stored consecutively, each beginning where the previous one
  ends: grid `n + 1` starts `mGridSize` bytes after grid `n`. A reader locates
  grid `n` by summing the preceding grids' `mGridSize`, which is how NanoVDB's
  own `GridHandle` indexes a multi-grid buffer.
- Grid `n` MUST record `mGridIndex` equal to `n`, and `n` MUST be the C-order
  (row-major, last `grid` dimension varying fastest) ravel of the chunk-local
  indices along the `grid` dimensions. This is what makes a grid addressable
  from array coordinates alone.
- `mGridName` is not authoritative: readers MUST resolve a grid by index, never
  by name, since nothing constrains names to be unique or populated. Writers
  MAY set names for the benefit of tools outside Zarr.

Each grid is an independent tree with its own topology, so a `grid` dimension
imposes no relationship between the active sets of successive indices — which
is what makes it the right role for a time axis. The per-array uniformity of
requirement 5 still applies to all of them.

### Grid coherence across chunks

Encoding each chunk as an unrelated grid — its own origin, alignment, and tree
shape — would leave readers unable to treat the chunks as parts of one
hierarchy. The following requirements prevent that. An array using this codec
MUST satisfy all of them, and each is checkable from the array metadata and
chunk buffers alone.

1. **One index space.** Each chunk's grids MUST express voxel coordinates in
   the global index space of this codec's input (the `z`, `y`, `x` indices
   `[k, j, i]` occupy NanoVDB coordinate `(i, j, k)` regardless of which chunk
   holds them); grids MUST NOT be rebased to a chunk-local origin. Chunks of a
   region then compose by union, and a writer can produce them by splitting one
   global grid. A preceding [`transpose`](../transpose/README.md) composes a
   constant permutation onto this mapping without weakening it: the array
   coordinate is recovered from the NanoVDB coordinate by `order`, which is
   metadata rather than per-chunk state.

2. **Leaf-aligned chunk grid.** The chunk extent along each of the `z`, `y`
   and `x` dimensions, and the chunk grid origin in each of them, MUST be an
   integer multiple of the leaf extent `1 << tree_config[2]` (8 for
   `[5, 4, 3]`), so that no leaf node straddles a chunk boundary.

3. **Node-aligned chunk shape (recommended).** The `z`, `y` and `x` chunk
   extents SHOULD additionally be an integer multiple of, or an integer divisor
   of, the lower internal node extent
   `1 << (tree_config[1] + tree_config[2])` (128 for `[5, 4, 3]`). A chunk that
   is an exact node extent is a clean subtree with a dense top level,
   addressable without searching the root node's tile table; other shapes
   satisfying requirement 2 remain conformant at the cost of a root-level
   lookup per traversal.

   Requirements 2 and 3 constrain only the three spatial dimensions. A
   `channel` dimension is governed by requirement 6, and a `grid` dimension by
   requirement 7; neither has any alignment constraint, because neither is part
   of the tree's index space. Where a `transpose` precedes this codec, the
   extents these requirements constrain are those of the array dimensions
   carrying the `z`, `y` and `x` roles — `chunk_shape[order[i]]` for the
   relevant `i` — since `transpose` permutes extents without changing them.

4. **Disjoint coverage.** Each of a chunk's grids MUST contain active voxels
   only within that chunk's spatial bounds, so for every index along the `grid`
   dimensions the array's active set is the disjoint union of its chunks'
   active sets.

5. **Uniform configuration.** Every grid of every chunk of an array MUST use
   the same `grid_type`, `tree_config`, `stats`, and background value, letting
   a reader specialize its traversal (including compiling a shader) once per
   array. This is what keeps a `grid` dimension cheap: the grids differ in
   topology and values, never in layout.

6. **Whole vectors per chunk.** The chunk extent along a `channel` dimension
   MUST equal the array extent along it, so that no chunk holds a partial
   vector.

7. **Grids addressable by index.** A chunk's grid count MUST equal the product
   of its extents along the `grid` dimensions, and grid `n` of the buffer MUST
   correspond to the C-order ravel of the chunk-local `grid` indices, as
   specified in [Grid enumeration](#grid-enumeration). A reader can then map an
   array coordinate to a grid without inspecting grid names or decoding any
   other chunk.

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

[`transpose`](../transpose/README.md) is the one `array -> array` codec that may
precede this one. No other may, and implementations SHOULD reject such a chain
when the array metadata is parsed.

### Why `transpose` is permitted

`transpose` is a pure permutation of dimension indices: rank-preserving,
exactly invertible, and declared in full by its `order`. It therefore does not
weaken any [grid coherence](#grid-coherence-across-chunks) requirement — the
tie between array indices and NanoVDB coordinates is composed with a constant
permutation rather than broken — and it is what lets a strict role order
coexist with whatever dimension order an array wants; see
[`dimension_roles`](#dimension_roles).

The concern that applies to `array -> array` codecs in general is that they
decode _after_ the `array -> bytes` codec, so a reader taking the pass-through
path never materializes a dense array for them to act on. For `transpose` this
costs nothing: the permutation is a per-array constant, so such a reader folds
it into its coordinate mapping instead of applying it to data. This is
normative, not merely an option — see
[Reader pass-through](#reader-pass-through).

### Why the others are not

1. [`reshape`](../reshape/README.md) splits and merges dimensions, preserving
   only C-order element order. Dimension identity does not survive it, so there
   is no dimension for a role to describe and requirement 1 has nothing to tie
   NanoVDB coordinates to. Combining it with `transpose` does not help: the
   composed operation is still not a permutation.
2. Value-domain codecs such as [`cast_value`](../cast_value/README.md) and
   [`scale_offset`](../scale_offset/README.md) transform every voxel. Unlike a
   permutation these cannot be folded into a coordinate mapping, so permitting
   them would forfeit the pass-through path for exactly the arrays this codec
   serves — and would leave a grid whose values and background do not match the
   array's `fill_value`. Their effect belongs in how the array was written.

`bytes -> bytes` codecs compose freely and are the right place for general
compression and checksums.

## Reader pass-through

Because the encoded buffer is already a pointer-free spatial index, a reader
MAY keep it rather than decode it to a dense array — including handing it to a
GPU as-is and traversing it in a shader. The coherence requirements make this
safe without inspecting the whole array: traversal is specialized once per
array (requirement 5), chunks are addressed without coordinate arithmetic (1),
no leaf spans a chunk boundary (2), and the root-level search can be skipped
when requirement 3 holds. Such readers benefit from `stats` being `minmax` or
`all`, which enables skipping subtrees outside the value range of interest;
encoders targeting them SHOULD write those rather than `none`.

Taking this path means skipping the `array -> array` decode stage, which would
otherwise have undone a preceding [`transpose`](../transpose/README.md). A
reader that does so MUST compose that permutation into its own coordinate
mapping, so that a coordinate in the array's dimension order still resolves to
the same voxel it would have through a full decode. Since `order` is a
per-array constant this is a relabelling of axes and not work per voxel, but
omitting it silently returns transposed results.

Everything else in this section is non-normative.

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

A segmentation mask, storing topology only:

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

Thresholded fluorescence, quantized and sparsified with a recorded tolerance:

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

A sparse `float32` displacement field shaped `[Z, Y, X, C]` with `C = 3`,
encoded as a `Vec3f` grid; the `channel` dimension is innermost and covered in
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

A time series of scalar volumes shaped `[T, Z, Y, X]`. The leading dimension
takes the `grid` role, so each chunk's buffer holds four grids -- one per
timepoint -- whose topologies are unrelated to each other. Nothing here is
specific to time: the same layout serves an ensemble index or a set of
independent fields, and `dimension_names` is where the meaning is recorded.

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

The same volumes with a three-component vector value per voxel, shaped
`[T, Z, Y, X, C]`: `grid` outermost, `channel` innermost, and the chunk covers
the `channel` dimension in full.

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

An OME-Zarr style array shaped `[T, C, Z, Y, X]` with `C = 3`, whose three
components belong to one voxel. A `transpose` moves the channel axis innermost
so that this codec's input is `[t, z, y, x, c]`; each timepoint becomes one
`Vec3f` grid. Note that `dimension_names` is in the array's order while
`dimension_roles` is in the transposed order, and `order` relates them:

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

- The encoded buffer is a plain NanoVDB grid, readable by NanoVDB and any tool
  built on it, independently of Zarr — once the `bytes -> bytes` stage has been
  undone. Since compression is both recommended and effective here, a stored
  chunk is generally not a NanoVDB buffer as it sits: reaching one means
  running the Zarr decode chain, or writing the array with no `bytes -> bytes`
  codec.
- Grids can be produced from OpenVDB grids with `nanovdb::createNanoGrid`, and
  from dense arrays with NanoVDB's build tools. OpenVDB's own `.vdb`
  serialization is _not_ interchangeable with a NanoVDB buffer and must be
  converted.
- NanoVDB buffers carry a version and declare their grid type, so a reader can
  detect a buffer written by a newer NanoVDB, or holding a value type it does
  not handle, and fail cleanly. They are not self-describing about the tree
  configuration, which is why [`tree_config`](#tree_config) is recorded in the
  array metadata.
- Reference implementation: in progress, targeting a Zarr reader that passes
  the buffer through to a GPU without densifying it.

## Change log

No changes yet.

## Current maintainers

- TBD
