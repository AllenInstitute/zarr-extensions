# warpfield codec

**Status: experimental draft.** The format may change without notice.

`warpfield` is an `array -> bytes` codec for **displacement (registration)
fields**. It is lossy, but every decoded sample is guaranteed to be within an
absolute `error_bound` of the original. That includes samples on tears, folds
and other step discontinuities, which defeat transform coders.

Fields are usually smooth with occasional sharp features. Arrays are chunked
in space, and images are pushed through the decoded field. The codec is built
around three further guarantees:

1. **No chunk seams.** Every sample on a chunk face is coded with a tighter
   `face_error_bound`. The jump in reconstruction error between the last sample
   of one chunk and the first of the next is therefore at most
   `2 * face_error_bound` (ε/2 with the default). On noisy fields that is
   statistically indistinguishable from the jumps inside a chunk. No chunk ever
   needs its neighbours to encode or decode.
2. **Invertibility (optional).** With `jacobian` configured, the encoder
   promotes samples to tighter bounds until no cell inside the chunk that
   satisfied `det(I + ∇u) > min_det` before compression violates it after, and
   no cell's determinant changes sign.
3. **Stable recompression.** Prediction is closed-loop on decoded values, so
   without `jacobian`, decode → encode → decode is a fixed point.

The design follows the error-bounded, interpolation-predictor compressor
family (SZ3, QoZ, HPEZ) and adds face classes, ENO stencils and Jacobian
promotion.

## Codec name

The value of the `name` member in the codec object MUST be `warpfield`.

## Configuration parameters

### `axis_roles`

Required. An array with one token per dimension of the codec's input (after any
preceding `array -> array` codec), outermost first.

| role          | meaning                                                                                           |
| ------------- | ------------------------------------------------------------------------------------------------- |
| `spatial`     | predicted across; part of the Jacobian                                                            |
| `smooth`      | predicted across, but not part of the Jacobian (e.g. time, or a section index along which fields vary smoothly) |
| `independent` | not predicted across: each index is coded on its own (e.g. timepoints with unrelated fields)        |
| `component`   | the vector component index; at most one; each component is coded on its own                       |

At least one dimension MUST be `spatial` or `smooth`. Typical layouts:

| field                                     | shape            | `axis_roles`                                                |
| ----------------------------------------- | ---------------- | ----------------------------------------------------------- |
| 2D section alignment                      | `[Y, X, 2]`      | `spatial, spatial, component`                               |
| 2D section fields stacked into a volume   | `[Z, Y, X, 2]`   | `smooth` (or `independent`), `spatial, spatial, component`  |
| 3D volume                                 | `[Z, Y, X, 3]`   | `spatial, spatial, spatial, component`                      |
| 3D fields over time, unrelated per time   | `[T, Z, Y, X, 3]`| `independent, spatial, spatial, spatial, component`         |
| 4D field, smooth in time                  | `[T, Z, Y, X, 3]`| `smooth, spatial, spatial, spatial, component`              |

### `error_bound`

Required. A positive number. It is the absolute error bound, in the units of
the stored values (typically voxels of the moving image), for interior samples.

### `face_error_bound`

Optional, in `(0, error_bound]`. The default is `error_bound / 4`. It is the bound for face
samples: samples whose index along any `spatial` or `smooth` dimension is 0 or
N−1, where N is the chunk extent along that dimension. Writers SHOULD always
record it.

### `predictor`

Optional: `"linear"`, `"cubic"` or `"eno"`. The default is `"eno"`. See
[Prediction](#prediction).

### `boundary`

Optional: `"clamped"` (the default) or `"extrapolate"`. `"extrapolate"` is the
SZ3-style anchor lattice and exists only for comparison experiments. It is not
seam-free, and writers SHOULD NOT use it for data.

### `jacobian`

Optional object. When it is present, the encoder enforces the Jacobian
guarantee. It has no effect on decoding, beyond the tight class being
recorded in the chunk.

- `grid_spacing` (required): one positive number per `spatial` dimension, in
  order. It is the distance between adjacent field samples, in the units of
  the stored values.
- `min_det` (default `0`): the determinant floor to preserve.
- `tight_error_bound` (default `error_bound / 8`): the bound for promoted samples.

`jacobian` requires a `component` dimension whose extent equals the number of
`spatial` dimensions. Component `c` is the displacement along the `c`-th
`spatial` dimension.

## Supported chunk shapes

Any. A chunk MUST cover the `component` dimension in full.

## Supported data types

`float32` and `float64`. NaN and ±Inf are stored exactly.

## Guarantees

Let `x` be an input sample and `x̂` its decoded value.

- **Bound.** `|x − x̂| ≤ ε(x)`, where ε is `error_bound`, `face_error_bound`,
  `tight_error_bound` or 0 according to the sample's class. Decoders compute it in
  the array's data type. Non-finite samples decode exactly.
- **Seams.** For adjacent samples `a | b` in different chunks,
  `|(x̂_a − x_a) − (x̂_b − x_b)| ≤ 2·face_error_bound`.
- **Continuous bound.** Multilinear interpolation uses convex weights, so the
  interpolated decoded field is within the same bound of the interpolated
  original *everywhere*, not just at samples.
- **Gradients.** A finite difference over one cell changes by at most
  `(ε_a + ε_b) / grid_spacing`.
- **Jacobian.** With `jacobian` set, consider every cell whose 2^k corners lie in
  one chunk, with forward differences divided by `grid_spacing`. If the original
  `det(I + ∇u) > min_det`, the decoded determinant is also `> min_det`. If the
  original determinant is nonzero, its sign is preserved. A cell that straddles
  two chunks is not checked, because the encoder cannot see both sides; its
  determinant can move by at most the gradient bound computed with
  `face_error_bound`.
- **Round trips.** Store only the forward field and compute the inverse ψ̂ from
  the decoded field φ̂. Then `|φ(ψ̂(y)) − y| ≤ |φ̂(ψ̂(y)) − y| + error_bound`,
  where the first term on the right is the inversion tolerance. If the forward
  and inverse fields are compressed separately, the bound is instead
  `|ψ̂(φ̂(x)) − x| ≤ ε_ψ + Lip_∞(ψ)·ε_φ + |ψ(φ(x)) − x|`.

## Format and algorithm

### Canonical layout

The decoder reorders the chunk to a canonical array of shape `(B, P₁ … Pₖ)`.
First come the `independent` dimensions, then the `component` dimension, in
input order and flattened C-order into one batch axis `B`. Then come the
`spatial` and `smooth` dimensions in input order. Every index in the format is
a C-order linear index into this canonical array. Each batch row is predicted
only from itself.

### Error classes

Every sample has a class, with a bound ε and bin width `w = 2ε` (float64):

| class | name     | ε                                  |
| ----- | -------- | ---------------------------------- |
| 0     | interior | `error_bound`                      |
| 1     | face     | `face_error_bound`                 |
| 2     | tight    | `jacobian.tight_error_bound`, or `error_bound / 8` |
| 3     | verbatim | 0 (value stored exactly)           |

The decoder assigns each class as follows:

1. Start from class 0.
2. Set class 1 on every sample whose index along any predicted axis with
   extent N ≥ 2 is 0 or N−1.
3. Set class 2 on every index in the tight list.
4. Set class 3 on every index in the verbatim list.

### Level schedule

Along a predicted axis of extent N, the *known set at stride s* is:

- `clamped`: `{0, s, 2s, …} ∪ {N−1}`, restricted to indices below N
- `extrapolate`: `{0, s, 2s, …}`

Let `S` be the smallest power of two with `S ≥ max(Nᵢ − 1, 1)`.

1. **Anchors.** Visit the Cartesian product of `known(S)` on every axis. Each
   of these samples has prediction 0.
2. **Refinement.** For `h = S/2, S/4, …, 1`, and for each predicted axis `j` in
   order, visit the Cartesian product of the following index sets in C order:
   - along each axis `i < j`: `known(h)`
   - along axis `j`: `known(h) \ known(2h)`, the targets
   - along each axis `i > j`: `known(2h)`

   Skip the pass if the target set is empty.

Within a pass, visit samples in C order of the `(B, …)` block. Each target is
predicted from samples of `known(2h)` on its own axis-`j` line. Those samples
have all been decoded already, so a sample on a chunk face is predicted only
from samples on that same face.

### Prediction

For a target at position `t`, take the anchors of `known(2h)` nearest to it on
each side: `l2 < l1 < t < r1 < r2`. A missing `l2` takes the position of `l1`, a missing
`r1` takes `l1`, and a missing `r2` takes `r1`. Missing slots carry weight 0.

Candidate stencils use float64 Lagrange weights for the actual positions:

- linear `(l1, r1)`
- quadratic-left `(l2, l1, r1)`
- quadratic-right `(l1, r1, r2)`
- cubic `(l2, l1, r1, r2)`

The weight for node `i` is computed as `w = 1`, then, for each other node `j` in
slot order, `w = (w × (t − x_j)) / (x_i − x_j)`. A prediction is the sum
`w_l2·v_l2 + w_l1·v_l1 + w_r1·v_r1 + w_r2·v_r2`, accumulated left to right.

- `linear`: linear. Without `r1` (`extrapolate` mode only), use linear
  extrapolation from `(l2, l1)` if `l2` exists, otherwise `v_l1`.
- `cubic`: cubic if all four slots exist, otherwise the available quadratic,
  otherwise as `linear`.
- `eno`: as `cubic`, except when all four slots exist. Then let
  `dL = |f[l2,l1,r1]|` and `dR = |f[l1,r1,r2]|` be the absolute second divided
  differences, computed as `((f_c − f_b)/(x_c − x_b) − (f_b − f_a)/(x_b − x_a)) / (x_c − x_a)`,
  and let `τ = error_bound / h²`. If `dL ≤ 3·dR + τ` and `dR ≤ 3·dL + τ`, use
  cubic. Otherwise use quadratic-left if `dL ≤ dR`, else quadratic-right. This
  keeps a step outside the target's interval from contaminating the prediction.

If the prediction is not finite, it is 0.

### Quantization and reconstruction

For a sample of class < 3, the decoder reads the next code `q` and sets
`x̂ = cast(pred + q × w)`. The cast is to the array's data type, and the stored
reconstruction, used by later predictions, is that cast value widened to
float64. The encoder computes `q = rint((x − pred) / w)` (ties to even). If
`|x − x̂| > ε`, `x` is not finite, or `|q| ≥ 2⁵²`, the sample becomes class 3.
The decoder does not repeat this check; it only needs the class lists.

With `jacobian` set, the encoder decodes the chunk and finds the violating
cells (see [Guarantees](#guarantees)). It promotes each corner of those cells
one step: to class 2 if its bound would shrink, otherwise to class 3. It then
re-encodes, repeating until there are no violations. Promotion to class 3 is
forced after three rounds. The loop terminates because cells whose corners are
all verbatim reproduce the original determinant exactly.

### Encoded representation

All integers are little-endian.

| offset | size  | field                                                              |
| ------ | ----- | ------------------------------------------------------------------ |
| 0      | 4     | magic `"WFLD"`                                                     |
| 4      | 1     | format version, `1`                                                |
| 5      | 1     | rank R of the canonical array                                      |
| 6      | 2     | reserved, 0                                                        |
| 8      | 8·R   | canonical shape, uint64                                            |
| …      | 16    | byte lengths of sections 0–3, uint32                               |
| …      | 24    | counts: codes, tight indices, verbatim indices, uint64             |
| …      |       | section 0: codes `q` in visit order (class 3 samples skipped), zigzag-encoded, LEB128 varints, one zstd frame |
| …      |       | section 1: sorted tight indices, delta-coded (first delta from 0), LEB128, zstd frame |
| …      |       | section 2: sorted verbatim indices, same coding                    |
| …      |       | section 3: verbatim values in index order, raw data type, zstd frame |

A decoder MUST reject a chunk whose canonical shape disagrees with the chunk
shape and `axis_roles`, or whose code count is not consumed exactly.

## Interaction with other codecs

`warpfield` is an `array -> bytes` codec that already applies zstd, so it is
normally the whole codec chain. It also works as the inner codec of
`sharding_indexed`. A preceding `transpose` changes which dimensions
`axis_roles` refers to.

## Examples

A 3D field sampled every 16 voxels, with the inverse-consistency guarantee:

```json
{
  "name": "warpfield",
  "configuration": {
    "axis_roles": ["spatial", "spatial", "spatial", "component"],
    "error_bound": 0.25,
    "face_error_bound": 0.0625,
    "predictor": "eno",
    "jacobian": { "grid_spacing": [16, 16, 16], "min_det": 0.1 }
  }
}
```

More examples are in [`examples/`](examples).

## Measured behaviour (reference implementation)

The reference implementation has five synthetic presets: smooth fields with
true step discontinuities (slip cracks of 24 and 96 voxels, a 12-voxel opening
crack, and a 16-voxel fold in which the two sides overlap), a steep invertible
band, a continuous over-compression with det J < 0, and misaligned sections
where a stack axis exists. With ε = 0.25 voxel, chunks of 64³ (3D) or 100² (2D),
and `jacobian.min_det` = 0.1, the results are:

- Compression ratios are 38–82×.
- The seam ratio (mean boundary error jump ÷ mean interior jump) is 0.63–0.93,
  versus 1.8–2.8 for the same predictor with SZ3-style extrapolation and no face
  classes.
- No Jacobian violations remain inside chunks. At most 44 remain per preset on
  cells that straddle a chunk face, versus 21–1051 without the Jacobian pass.
- zfp fixed-accuracy at the same tolerance reaches 3–10×; loosened until its
  realised maximum error matches, 5–15×.

On real fields the results are as follows. The fields are a pair of ANTs SyN
displacement fields (forward and inverse) registering two mouse brain
templates: 25 µm grids, 1.4 and 2.8 GB of float32. They were coded in
64³ × 3 chunks with `jacobian.min_det` = 0.1.

- At ε = 2.5 µm (0.1 voxel) the stores are 485× and 739× smaller than the raw
  arrays, with no Jacobian violations. At ε = 1 µm and 6.25 µm the ratios are
  293–467× and 732–1099×.
- zfp, loosened until its realised maximum error matches ε, is 11–27× larger.
- These fields are much smoother than the synthetic ones, so the error inside
  a chunk varies slowly and the seam ratio is 1.3–2.4 (zfp: 1.6–2.1). The
  largest boundary jump stays at the guaranteed `2 * face_error_bound`. zfp's
  largest boundary jump exceeds its own maximum error.

## Implementations

- `warpfield` (Python, numpy + zstandard): the reference implementation of the
  format, usable without zarr (one array, or a large array tile by tile), with
  the synthetic data generator, benchmarks and image gallery.
- `zarr-warpfield` (Python): this codec for zarr-python 3, a thin plugin over
  `warpfield`.
- `warpfield` (Rust crate) in the same repository, with Python bindings
  (`warpfield-rs`) and a WebAssembly build with a TypeScript wrapper. All three
  emit the same quantization codes and decode to bit-identical values;
  decoding runs at 275–525 MB/s per core natively and 190–340 MB/s in wasm.
- Java bindings (JNI to the Rust crate, Java 8+, one jar for all platforms)
  in the same repository, also bit-identical, decoding at the native speed.
- Neuroglancer (branch `warpfield-transform`) decodes the codec with the
  WebAssembly build.

## Change log

- Draft 1: initial version.

## Current maintainers

- Forrest Collman (Allen Institute)
