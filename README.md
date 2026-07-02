# mach-font

Pure-[Mach](https://github.com/briar-systems/mach) TrueType parsing and glyph
rasterization: read a `.ttf`, walk its tables, and rasterize glyphs into
coverage bitmaps a renderer can pack into an atlas. No C, no FreeType — just
Mach algorithms over the font's bytes. Project id is `font`, so consumers reach
everything as `font.*`.

```mach
use font;

fun example(data: *u8, len: u64) {
    # every scalar in a truetype file is big-endian and bounds-checked on read
    var units_per_em: u16 = 0;
    if (font.read_u16(data, len, 18, ?units_per_em)) {
        # ... parse the table directory, load a glyph, rasterize
    }
}
```

Consuming projects vendor the library as a normal Mach dependency. There is no
system link requirement: `mach-font` is pure algorithms and declares no `libs`.

```toml
[deps.mach-font]
git = "https://github.com/briar-systems/mach-font"
ref = "branch/main"
```

## Scope

- TrueType parsing: the table directory and the core tables a rasterizer needs
  (`glyf`, `loca`, `head`, `maxp`, `cmap`, `hmtx`, ...).
- Glyph rasterization at [stb_truetype](https://github.com/nothings/stb)-level
  quality: quadratic outlines flattened and scan-filled to an anti-aliased
  coverage bitmap.
- Atlas-friendly output: glyph coverage plus the metrics (advance, bearings,
  bounding box) a renderer needs to lay glyphs into a texture atlas.

## Non-goals

- FreeType-parity hinting. The bytecode interpreter and grid-fitting are out of
  scope; rasterization is unhinted.
- Complex text shaping — bidi, cursive joining, contextual substitution
  (HarfBuzz territory). Single-run left-to-right layout for Latin and friends is
  enough initially.
- Font editing or subsetting. `mach-font` reads fonts; it does not write them.

## Platforms

`mach-font` has no OS dependencies — it operates entirely on in-memory bytes —
so it builds for every prime target the Mach compiler ships:
`x86_64`/`aarch64` on Linux and macOS, and `x86_64` on Windows. Endianness is
handled explicitly (TrueType is big-endian regardless of host), so nothing here
is host-specific.

## Layout

```
src/
  font.mach   library surface: re-exports the public api behind `use font;`
  read.mach   bounds-checked big-endian reads over the font's bytes — the
              shared foundation the table parsers and rasterizer build on
```

The reader is the first piece: font files are untrusted input, so every read
validates its span against the buffer before touching it and reports failure
rather than reading out of bounds. Table parsers and the rasterizer land on top
of it.

## Tests

Tests are display-free and allocation-free: they build small byte buffers in
place and assert the readers decode big-endian values, honor bounds, and match
table tags. Run them with `mach test .`.

## License

See [LICENSE](./LICENSE).
