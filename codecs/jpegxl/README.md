# jpegxl codec

Defines an `array -> bytes` codec that encodes an array chunk as a
[JPEG XL](https://jpeg.org/jpegxl/) codestream.

JPEG XL supports both lossless and lossy compression of 8- and 16-bit integer
and floating-point samples, with one or more channels and one or more frames.
This codec is intentionally minimal: it fixes a simple relationship between the
array chunk and the JPEG XL image, and delegates all dimension rearrangement to
the [`reshape`](../reshape/README.md) and [`transpose`](../transpose/README.md)
codecs.

> This document is a proposed extension. It is licensed under the
> [Creative Commons Attribution 3.0 Unported License](https://creativecommons.org/licenses/by/3.0/).

## Codec name

The value of the `name` member in the codec object MUST be `jpegxl`.

## Configuration parameters

The codec has no required configuration parameters; the `configuration` object
MAY be omitted or empty. Any members that are present are **encoder hints**: an
encoder MAY use them to control how it produces the codestream (for example the
compression `distance` or `effort`), and they serve as a record of how the
codestream was produced. They have no normative meaning for decoding.

The `configuration` object MUST not be utilized in decoding. A decoder MUST
derive the image geometry (`W`, `H`, `S`, `F`), the sample data type, and the
color encoding solely from the JPEG XL codestream, and:

- MUST NOT change its output based on any member of `configuration`;
- MUST NOT reject a chunk because `configuration` contains members it does not
  recognize.

This design is possible because a JPEG XL codestream fully self-describes its
dimensions, bit depth, and color encoding — including any internal color
transform (XYB, YCbCr) and chroma subsampling. There is therefore **no
decode-time parameter** analogous to the color-space selector that baseline
JPEG/JFIF requires (where three channels are ambiguously either RGB or YCbCr
and the decoder must be told which). 

Because the members are non-normative, `configuration` is an **open set**:
encoders MAY record any implementation-specific parameters — for example
`effort`, `distance`, `lossless`, `decodingspeed`, `photometric`, or
`bitspersample` — and a decoder MUST reconstruct the array independant of them. This lets an implementation faithfully record its full
encoder parameter set (aiding provenance and lossless re-encoding across a
decode/encode cycle) without affecting interoperability.

See [`schema.json`](./schema.json) for the documented members; additional
members are permitted.

## Encoded representation

The encoded chunk is a JPEG XL image in either of the two forms permitted by
the JPEG XL standard: a bare codestream (beginning with `0xFF 0x0A`) or the
ISOBMFF box container (beginning with the JXL container signature `0x00 0x00 0x00 0x0C 0x4A 0x58 0x4C 0x20 0x0D 0x0A 0x87 0x0A`). Decoders
MUST accept both forms. Encoders SHOULD write a bare codestream; the container
form exists to carry metadata boxes (Exif, XMP, JPEG-reconstruction data) that
this codec does not use, and decoders MUST ignore any such boxes.

## Array shape contract

Let the JPEG XL image have `width` (`W`), `height` (`H`), `samples` (`S`, the
number of interleaved sample channels) and `frames` (`F`, the number of
animation keyframes). The decoded array chunk MUST equal the _native JPEG XL
shape_

```
[F, H, W, S]
```

in C (row-major) order, with the `F` axis omitted when `F == 1` and the `S` axis
omitted when `S == 1`. Equivalently, the chunk shape MUST be one of:

| chunk shape    | meaning                      |
| -------------- | ---------------------------- |
| `[H, W]`       | single-frame, single-channel |
| `[H, W, S]`    | single-frame, multi-channel  |
| `[F, H, W]`    | multi-frame, single-channel  |
| `[F, H, W, S]` | multi-frame, multi-channel   |

Decoders MUST return an error if the chunk shape is not one of these forms, or
if `W`, `H`, `S`, `F` derived from the codestream are not consistent with it.

**Disambiguating the two 3-D forms.** A three-dimensional chunk is either
`[H, W, S]` (single frame, `S` channels) or `[F, H, W]` (`F` frames, one
channel). Following the convention of the reference bindings
([`imagecodecs`](https://github.com/cgohlke/imagecodecs)), the two are
distinguished by the size of the trailing axis: a trailing axis of size **≤ 4**
is the sample axis `S`, giving `[H, W, S]`; a trailing axis of size **≥ 5** is
the width `W`, so the leading axis is the frame axis `F`, giving `[F, H, W]`. A
consequence is that the interleaved-sample form `[H, W, S]` is limited to
`S ≤ 4` (see [Channels](#channels-s)); an array with more than four independent
channels is stored with the channel axis as a separate chunk/frame dimension,
not interleaved.

Apart from this trailing-axis rule for 3-D chunks (needed only because `[H,W,S]`
and `[F,H,W]` are both three-dimensional), the codec does not guess which array
dimensions are spatial, channel, or frame dimensions. To store an array whose
chunk shape is not already in one of the forms above, insert a `reshape` codec
(and, if the channel axis is not innermost, a `transpose` codec) before
`jpegxl`. This is the same division of responsibility used by other constrained
codecs, and it moves the "squeeze" behavior of some JPEG XL bindings into the
separate `reshape` codec. 

### Channels (`S`)

The JPEG XL codestream distinguishes _color channels_ (1 for grayscale or 3 for
a color image; XYB/YCbCr transforms and chroma subsampling apply only to these)
from _extra channels_ (alpha, depth, and other data), and the format permits a
large number of extra channels. The `S` axis of the decoded chunk is the total
number of interleaved sample channels the decoder produces (color channels plus
extra channels).

Because a trailing axis of size `≥ 5` denotes a spatial axis rather than the
sample axis (see the disambiguation rule above), the interleaved-sample form
`[H, W, S]` covers only `S ∈ {1, 2, 3, 4}`: `1`–`2` map to one grayscale color
channel (plus, for `2`, one extra channel) and `3`–`4` map to three RGB color
channels (plus, for `4`, one extra/alpha channel). In practice this codec's
reference decoder supports `S ∈ {1, 3, 4}` — grayscale, RGB, and RGBA — and
returns the RGBA alpha channel as stored (it is not forced opaque). Decoders
MUST return an error for a channel count they do not support rather than
silently mismatching the chunk shape.

`S` is **not** a general mechanism for stacking many independent measurement
channels. For data with an arbitrary number of independent channels (e.g.
fluorescence or multispectral microscopy), do not encode them as one
multi-channel image; instead make the channel axis a chunk/shard dimension
(using `reshape`/`transpose`) so each channel is compressed independently as a
grayscale `[H, W]` (or `[H, W, 1]`) image. This both fits the supported channel
counts and preserves per-channel fidelity (see below). If multi-channel images are natural colorized images in which it would make sense to combine 3 channels together then one can utilize a [H,W,3] compression format, but one should choose to do so explicitly and not default to it.

## Supported data types

`uint8`, `uint16`, and `float32`.

The sample bit depth recorded in the codestream's image header MUST match the
array data type: `bits_per_sample` of 8 for `uint8`, 16 for `uint16`, and
32-bit float samples (`bits_per_sample` 32 with 8 exponent bits) for
`float32`. Decoders MUST return an error on a mismatch rather than rescale
samples to the range of the array data type — for example, a 12-bit
codestream MUST NOT be expanded to the full `uint16` range, since the
rescaled values would silently differ from the values originally stored in
the array.

## Sample layout and color

Samples are stored in C order, so for a chunk shape ending in `S` the channels
are interleaved (the innermost, unit-stride axis). If a different in-memory
channel order is required, use the `transpose` codec.

Decoders MUST return samples in the color space signaled by the codestream's
image header, inverting any codestream-internal transforms (XYB, YCbCr,
chroma upsampling) as defined by the JPEG XL specification, and MUST NOT
apply any further conversion toward a perceptual or display color space (for
example forcing an sRGB gamma or applying an ICC-profile conversion). Note
that many general-purpose JPEG XL APIs convert to a preferred or display
profile by default; implementations of this codec must disable that. When
JPEG XL is used losslessly, the decoded samples are bit-exact copies of the
encoded array.

Unlike the JPEG codec, this codec has **no** `encoded_color_space` or
`subsampling` parameters: JPEG XL manages color internally within the
codestream. 

> **Note:** JPEG XL can be lossy. Repeated decode/encode cycles compound
> artifacts, and lossy compression is unsuitable for label/segmentation data.

## Examples

### 2-D grayscale tile

A single-channel 2-D tile is a `[H, W]` chunk encoded directly, with no
`reshape` needed.

```json
{
  "chunk_grid": {
    "name": "regular",
    "configuration": { "chunk_shape": [256, 256] }
  },
  "codecs": [{ "name": "jpegxl", "configuration": {} }]
}
```

### 2-D RGB tile

A natural-color image with the three color channels interleaved as the innermost
axis is a `[H, W, 3]` chunk. The trailing axis of size 3 is the sample axis, so
JPEG XL encodes an RGB image. Choose this deliberately for true-color data; do
not use it to stack unrelated channels (see [Channels](#channels-s)).

```json
{
  "chunk_grid": {
    "name": "regular",
    "configuration": { "chunk_shape": [256, 256, 3] }
  },
  "codecs": [{ "name": "jpegxl", "configuration": {} }]
}
```

### RGBA (color + alpha)

A `[H, W, 4]` chunk encodes three RGB color channels plus one alpha (extra)
channel. The alpha channel is stored and returned as-is (not forced opaque).

```json
{
  "chunk_grid": {
    "name": "regular",
    "configuration": { "chunk_shape": [256, 256, 4] }
  },
  "codecs": [{ "name": "jpegxl", "configuration": {} }]
}
```

### Channel axis not innermost (`transpose`)

If the color channel axis is not the innermost dimension — for example a
channel-first `[3, H, W]` chunk (`c, y, x`) — insert a `transpose` codec to move
it to the innermost position, so `jpegxl` receives `[H, W, 3]`.

```json
{
  "chunk_grid": {
    "name": "regular",
    "configuration": { "chunk_shape": [3, 256, 256] }
  },
  "codecs": [
    { "name": "transpose", "configuration": { "order": [1, 2, 0] } },
    { "name": "jpegxl", "configuration": {} }
  ]
}
```

### Multi-frame stack (`reshape`)

The metadata below stores a `[1, 1, 32, 256, 256]` chunk (for example a
`c, t, z, y, x` layout with unit `c` and `t`) as a single 32-frame JPEG XL
image. The `reshape` codec drops the two leading unit dimensions to produce the
`[32, 256, 256]` (`[F, H, W]`) native image shape.

```json
{
  "chunk_grid": {
    "name": "regular",
    "configuration": { "chunk_shape": [1, 1, 32, 256, 256] }
  },
  "codecs": [
    { "name": "reshape", "configuration": { "shape": [[2], [3], [4]] } },
    { "name": "jpegxl", "configuration": {} }
  ]
}
```

### Many independent channels (compress each separately)

For data with many independent measurement channels (e.g. fluorescence or
multispectral microscopy), do not interleave them as `S`. Chunk the channel axis
to 1 so each chunk holds a single channel, then use `reshape` to drop that unit
axis, encoding each channel as an independent grayscale image. Here a
`[c, z, y, x]` array chunked `[1, 32, 256, 256]` becomes a `[32, 256, 256]`
grayscale frame stack, one per channel.

```json
{
  "chunk_grid": {
    "name": "regular",
    "configuration": { "chunk_shape": [1, 32, 256, 256] }
  },
  "codecs": [
    { "name": "reshape", "configuration": { "shape": [[1], [2], [3]] } },
    { "name": "jpegxl", "configuration": {} }
  ]
}
```

### Lossy encoding with encoder hints

Encoder parameters are recorded in `configuration` purely as hints; a decoder
reconstructs the array from the codestream regardless of them. Here a `[H, W]`
tile is encoded lossily at Butteraugli `distance` 1.0.

```json
{
  "chunk_grid": {
    "name": "regular",
    "configuration": { "chunk_shape": [256, 256] }
  },
  "codecs": [
    {
      "name": "jpegxl",
      "configuration": { "lossless": false, "distance": 1.0, "effort": 7 }
    }
  ]
}
```

## Example data

See the fixtures under
[`testdata/jxl`](https://github.com/google/neuroglancer/tree/master/testdata/jxl)
in the neuroglancer repository (`gray_u8_4x4.jxl`, `rgb_u8_2x2.jxl`, and the
1×1 `uint8`/`uint16`/`float32` samples), together with their `fixtures.json`
describing the expected decoded values.

## Interoperability and compatibility

- The native shape contract matches the (squeezed) decode output of the
  [`imagecodecs`](https://github.com/cgohlke/imagecodecs) `JpegXl` numcodecs
  codec, with the `squeeze` behavior delegated to the `reshape` codec.
- A reference decoder implementation is provided by
  [neuroglancer](https://github.com/google/neuroglancer) in
  `src/datasource/zarr/codec/jpegxl`, which decodes  `jpegxl` chains using the `jxl-oxide` JPEG XL decoder compiled to WebAssembly.  It also supports the `transpose` and `reshape` codecs which are required to make this practically useful for general Nd data. 

## Change log

No changes yet.

## Current maintainers

- TBD