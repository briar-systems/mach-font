# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Changed
- **Breaking: the library surface moves from `src/font.mach` to `src/lib/font.mach`, so its full module path is now `font.lib.font` instead of `font.font`** (#57). The `[artifact.font]` entry follows the family layout, where artifact entries sit under `src/lib/` or `src/bin/`. A bare `use font;` is unaffected, since it binds the default artifact's entry wherever that lives, and every other module path (`font.glyf`, `font.info`, `font.raster` and the rest) is unchanged. Only a consumer that imports the surface by its full path, `use font.font;`, must change it to `use font.lib.font;` or to the bare `use font;`. `mach test . --list` collects the same 106 tests as before.

## [0.8.0] - 2026-09-25

### Changed
- **Breaking: builds against std 8.0.0 and requires mach 5.12** (#53). `[dep.std]` moves from `^6.0` to `^8.0`, realized to v8.0.0 by the committed `dep/std` gitlink, and `[project].mach` rises from `^5.9` to `^5.12`, which std 8 requires. Resolution is flat, so a consumer of mach-font must move to std 8 and mach 5.12 with it, and must rebuild anything that links std rather than only recompiling against the new sources. mach-font imports only `std.runtime` and `std.types`, none of which std 7 or 8 changed, so no source changed. Every module that holds a test is reached from `font.mach`, so mach 5.12's closure-scoped `mach test .` (briar-systems/mach#3813) still collects all 106 tests on every target.
- ci: the lib job seeds mach v5.12.0 until the family pin moves (briar-systems/.github#103) (#53).

## [0.7.0] - 2026-09-19

### Changed
- deps: std moves to `^6.0` (pinned at v6.0.0), and the compiler floor rises to `mach = "^5.9"`. std 6.0.0 reshaped sort, heap, map/set, ct and buffers. mach-font imports only `std.runtime` and `std.types`, so no source changes.

## [0.6.1] - 2026-09-19

### Changed
- deps: std is declared by version range (`^5.7.1`) rather than an exact tag, with the realized `dep/std` gitlink as the pin. A root project that declares std by range no longer conflicts with this library on the next std minor.

## [0.6.0] - 2026-09-19

### Changed
- deps: std moves to `tag/v5.7.1`, so a root project on std 5.7.x can override this library's std without breaking it. std 5.7.1 requires mach 5.5.2, so the manifest's compiler range rises to `^5.5.2`. mach-font imports only `std.runtime` and `std.types`, none of which std 5 reshaped, so no source changes.

## [0.5.0] - 2026-09-18

### Changed
- api: A glyph's bounding box comes from its resolved outline, through the new
  `glyf.outline_bounds` (re-exported as `font.outline_bounds`), never from the
  `glyf` header. `glyph_info` now takes the same scratch buffers as
  `glyph_outline` and returns `res[opt[glyf.Bounds], FontError]`, none for a
  glyph that resolves to no points. `render_glyph` places against that same box.
  `glyf.GlyphInfo` is renamed `glyf.GlyphHeader`, since `glyph_header` is the
  only thing that still returns it. Consumers: for a font whose stored boxes are
  correct (DejaVu, and every font in a 913-file scan) the measured box is
  identical to the header's, so nothing moves for shipped fonts. For a font whose
  composite box is stale or zero, placement changes from wrong to right (#9).
- ci: The tag-triggered workflow is `cd.yml`, and it serialises runs per tag so a doubled tag push cannot publish twice.
- build: The manifest declares the compiler range `mach = "^5.3"`, so mach 5.3 and later no longer warn about a missing range.
- license: Copyright is attributed to Briar Systems LLC.
- ci: Releases publish through the family release workflow (`briar-systems/.github` `mach-release.yml`). Pushing a `v*` tag verifies the tag against the manifest version and changelog, runs every CI leg, and publishes the GitHub release with the changelog section as notes.

## [0.4.1] - 2026-09-16

### Changed
- deps: std moves to `tag/v4.0.0`, which needs mach 5.2.0 or later. mach-font uses none of the std APIs that 4.0.0 removed or reshaped, so no source changes.

## [0.4.0] - 2026-09-16

### Changed
- deps: std moves to `tag/v3.2.0`. mach-font uses no std io, so the io runtime changes in std 3.x need no source changes.
- ci: CI runs the family pipeline (`briar-systems/.github` `mach-lib.yml`) on the pinned, checksum-verified mach seed: debug and release build and test, `mach fmt --check` and an all-targets release build on x86_64-linux for pull requests into dev, plus native aarch64-linux, windows and darwin legs for pull requests into main. A `gate` job is the one required check.
- build: Moved to Mach 5.0 and std 2.1. The manifest states every profile in
  full and marks its defaults, std is pinned as the `dep/std` gitlink at
  `tag/v2.1.0` under the `std` project id, and `mach.lock` is gone.
- api: Every fallible operation reports a `FontError` (new in `font.error`,
  re-exported as `font.FontError`) with one case per failure, in place of a
  `bool` success flag and out parameters. Results come back as
  `res[T, FontError]`, `err[FontError]`, or an `opt` inside the `res` where a
  font may simply lack something:
  - `read_u8`/`read_u16`/`read_i16`/`read_u32`/`read_f2dot14` return
    `res[T, FontError]`, `tag_eq` returns `res[bool, FontError]`.
  - `table.is_truetype` is replaced by `check_truetype(data, len) err[FontError]`,
    which reports the sfnt version it refused. `num_tables` returns a `res`, and
    `find_table` returns `res[opt[Table], FontError]`, none when the directory
    lacks the tag.
  - `parse_head`, `parse_hhea`, `num_glyphs` return a `res`. `parse_head` refuses
    a zero units-per-em. `storage_maxima` returns `res[opt[Maxima], FontError]`,
    none for a version 0.5 maxp.
  - `hmtx.metrics` returns `res[HMetrics, FontError]` (`advance`, `lsb`).
  - `loca.glyph_range` returns `res[GlyphRange, FontError]` (`start`, `end`), and
    `loca.is_empty` tests a range.
  - `glyf.glyph_header`, `point_count`, `extract_simple` return a `res`.
    `read_component` returns `res[Component, FontError]`, with the offset of the
    next record in the new `Component.next` field. `point_count` of an empty
    glyph is 0.
  - `cmap.select_subtable` and `cmap.lookup` return a `res`.
  - `kern.select_subtable` returns `res[Pairs, FontError]` (re-exported as
    `KernPairs`), and `kern.lookup` takes that `Pairs` in place of an offset and
    count.
  - `raster.flatten` returns `res[usize, FontError]`, the edge count.
  - `info.init` returns `res[Font, FontError]`. `Font` holds the parsed `head`,
    `hhea` as `opt[Hhea]`, the `hmtx`, `loca` and `glyf` offsets as `opt[usize]`,
    and the cmap and kern subtable selections as `res` values that keep the reason
    a font has no usable mapping or kerning. The flat `units_per_em`,
    `index_to_loc_format`, `ascent`, `descent`, `num_h_metrics`, `head_off`,
    `hhea_off`, `cmap_off`, `cmap_sub`, `kern_off`, `kern_pairs` and
    `kern_num_pairs` fields are gone.
  - `glyph_hmetrics`, `glyph_kern_advance`, `glyph_index`, `glyph_outline`,
    `outline_maxima` return a `res`, and `render_glyph` returns `err[FontError]`.
  - `scale_for_pixel_height` returns `res[f32, FontError]` in place of a 0.0
    sentinel. `scale_for_em_size` still returns `f32`, since `init` refuses a
    zero units-per-em.

### Added
- manifest: `linux-arm64` and `darwin-aarch64` targets, so the native aarch64 hosts build and test for themselves instead of falling back to linux-x86_64.
- info: `glyph_info(f, glyph)` reads a glyph's header, none for an empty glyph,
  so a consumer no longer resolves loca and glyf itself.

## [0.3.0] - 2026-08-09

### Added
- glyf: Composite glyph component records are decoded, so a glyph built from
  other glyphs resolves to its outline instead of coming back empty. Accented
  Latin is composite in most fonts, so this is the difference between rendering
  a language and rendering only ASCII.
- kern: Format 0 horizontal kerning pairs are read, and `font_info` surfaces
  them alongside the resolved composite outlines.

### Changed
- manifest: Re-touched to RFC-exact totality per mach#1964/mach#1979.

## [0.2.0] - 2026-07-07

Overhauls the build manifest to comply with the v2 build system schema.

### Changed
- manifest: Migrated manifest to v2 schema (`[artifact.font]`).
- deps: Updated `mach-std` dependency to the git URL.

## [0.1.0] - 2026-07-02

### Added
- Initial release of mach-font.
