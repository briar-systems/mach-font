# mach-font

Pure-[Mach](https://github.com/briar-systems/mach) TrueType parsing and glyph
rasterization: read a `.ttf`, walk its tables, and rasterize glyphs into
coverage bitmaps a renderer can pack into an atlas. No C, no FreeType, just
Mach algorithms over the font's bytes. Project id is `font`, so consumers reach
everything as `font.*`.

```mach
use font;
use std.types.result.res;
use std.types.size.usize;

fun example(data: *u8, len: usize) u16 {
    val face: res[font.Font, font.FontError] = font.init(data, len);
    if (sel face.err) {
        # face.err names the fault: truncated, not_truetype, missing_table, ...
        ret 0;
    }
    var f: font.Font = face.ok;
    val glyph: res[u16, font.FontError] = font.glyph_index(?f, 'A');
    if (sel glyph.ok) { ret glyph.ok; }
    ret 0;
}
```

Consuming projects vendor the library as a normal Mach dependency. There is no
system link requirement: `mach-font` is pure algorithms and declares no `libs`.

```sh
mach dep add . font --git https://github.com/briar-systems/mach-font --ref branch/main
```

## Scope

- TrueType parsing: the table directory and the core tables a rasterizer needs
  (`glyf`, `loca`, `head`, `maxp`, `cmap`, `hmtx`, `kern`, ...), including
  composite glyphs resolved into a single outline.
- Glyph rasterization at [stb_truetype](https://github.com/nothings/stb)-level
  quality: quadratic outlines flattened and scan-filled to an anti-aliased
  coverage bitmap.
- Atlas-friendly output: glyph coverage plus the metrics (advance, bearings,
  bounding box, pair kerning) a renderer needs to lay glyphs into a texture
  atlas.

## Non-goals

- **TrueType hinting.** The bytecode interpreter and grid-fitting are out of
  scope; rasterization is unhinted. Unhinted anti-aliased rendering is the modern
  norm (every current desktop and mobile stack ignores the hints at the sizes
  that matter), so the interpreter would buy nothing here.
- **`GSUB`/`GPOS` shaping.** Ligature and contextual substitution, mark
  attachment, and OpenType positioning are HarfBuzz territory, as are bidi and
  cursive joining. Single-run left-to-right layout with `kern` pair spacing is
  what this library supports.
- **CFF/OTF outlines.** `mach-font` reads `glyf` (quadratic) outlines only;
  `OTTO` files, whose cubic outlines live in a `CFF ` table, are rejected at
  `init` rather than partially parsed.
- Font editing or subsetting. `mach-font` reads fonts; it does not write them.

## Platforms

`mach-font` has no OS dependencies, since it operates entirely on in-memory
bytes, so it builds for every prime target the Mach compiler ships:
`x86_64`/`aarch64` on Linux and macOS, and `x86_64` on Windows. Endianness is
handled explicitly (TrueType is big-endian regardless of host), so nothing here
is host-specific.

## Layout

```
src/
  lib/
    font.mach   library surface and artifact entry: re-exports the public api
                behind `use font;`
  error.mach    FontError, one case per way an operation can fail
  read.mach     bounds-checked big-endian reads, the shared foundation
  table.mach    sfnt offset table + table directory lookup by tag
  head.mach     font header: unitsPerEm, loca format, bounding box
  maxp.mach     glyph count and outline storage maxima
  hhea.mach     horizontal header: ascent, descent, long-metric count
  hmtx.mach     per-glyph advance width and left-side bearing
  cmap.mach     character-to-glyph mapping (format 4 and format 12)
  kern.mach     pair kerning (format 0 horizontal)
  loca.mach     glyph location table (short and long offsets)
  glyf.mach     glyph outlines: point extraction + composite component records
  raster.mach   outline flattening + coverage-bitmap fill
  info.mach     the Font facade tying the tables together
```

The reader is the foundation: font files are untrusted input, so every read
validates its span against the buffer before touching it and reports
`FontError.truncated` rather than reading out of bounds. The table parsers and
rasterizer are built on top of it and follow the same discipline, so a truncated
or malformed font is rejected cleanly.

Every fallible call returns `res[T, FontError]`, `err[FontError]` or, where a
font may simply lack something, an `opt` inside the `res`. `FontError` has one
case per failure, carrying the offset, version, glyph index or table that
identifies it, so a caller can tell a broken file from an unsupported feature
from a buffer it sized too small.

## Status

Parsing covers the table directory, global metrics (`head`/`maxp`/`hhea`/`hmtx`),
character mapping (`cmap` formats 4 and 12), pair kerning (`kern` format 0), and
outline extraction (`loca` + `glyf`) for both simple and composite glyphs.
Rasterization flattens quadratic outlines and scan-fills an anti-aliased 8-bit
coverage bitmap with the nonzero winding rule. The `Font` handle (`font.init`)
locates the tables once and exposes `glyph_index`, `glyph_hmetrics`,
`glyph_kern_advance`, `glyph_outline`, `glyph_info`, `outline_maxima`, and
`render_glyph`. All
rasterization buffers are caller-provided, so the library allocates nothing.

A composite glyph resolves recursively into one flat outline, so `render_glyph`
rasterizes an accented letter exactly as it does a simple one. Size the point and
contour buffers from `outline_maxima`, since a composite's resolved point count
exceeds anything `point_count` reports for a simple glyph.

A glyph's bounding box is measured from that resolved outline, never read from
the `glyf` header. `glyph_info` reports it and `render_glyph` places against it,
both through `outline_bounds`, so a bitmap sized from `glyph_info` fits what
`render_glyph` draws. For a font whose stored boxes are correct the numbers are
identical to the header's. For a font whose composite box is stale or zero, the
glyph lands where its components actually are.

Known limitations of this pass:

- **Point-matched components are rejected.** A component placed by matching a
  point index against the glyph built so far, rather than by an explicit offset,
  is refused instead of being silently misplaced. It is vanishingly rare in
  shipping fonts.
- **The fill is unoptimized.** Coverage is sampled on a supersampling subgrid
  per pixel. That is correct and simple, but a sorted active-edge sweep would be faster
  for large glyphs.

## Tests

Tests are display-free and allocation-free: they build small byte buffers in
place and assert the readers decode big-endian values, honor bounds, and match
table tags. Run them with `mach test .`.

## License

See [LICENSE](./LICENSE).
