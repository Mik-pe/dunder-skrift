# dunder-skrift

A pure-Dunder, FreeType-inspired font reader, shaping/positioning helper, variable font evaluator, and analytic anti-aliased scanline rasterizer — zero external C dependencies.

## Features

- **Containers**: Raw SFNT (`0x00010000`, `OTTO`, `true`, `typ1`), TrueType Collections (`ttcf`, multi-face selection), and **WOFF1** (`wOFF` with pure-Dunder RFC 1950/1951 zlib Inflate supporting stored, fixed Huffman, and dynamic Huffman DEFLATE blocks + Adler-32 checksum verification). Clear diagnostic errors on `WOFF2` and malformed inputs.
- **Core OpenType/TrueType Tables**: `head`, `maxp`, `hhea`, `hmtx` (including trailing monospaced advance reuse), `OS/2`, `post`, `name` (UTF-16BE including surrogate pairs + MacRoman decoding).
- **Character Mapping (`cmap`)**: Formats `0`, `4` (both `idDelta` and `idRangeOffset`), `6`, `12` (full Unicode `U+0000..U+10FFFF`), and `14` (Unicode Variation Sequences).
- **Outlines (`glyf`/`loca`, `CFF `, `CFF2`)**:
  - `glyf` + short/long `loca`: simple contours, flag repeat compression, implicit quadratic Bezier midpoints, and compound/composite glyphs with translation, scale, and 2×2 affine transforms.
  - `CFF` Type 2 charstrings: global/local subroutines (`callgsubr`/`callsubr`/`return`), width parsing, `hintmask`/`cntrmask`, `rrcurveto`/`hhcurveto`/`vvcurveto`/`hvcurveto`/`vhcurveto`, `rcurveline`/`rlinecurve`, `hflex`/`flex`/`hflex1`/`flex1`, and `FDArray`/`FDSelect` (Formats 0, 3, 4).
  - `CFF2`: variable Type 2 outlines with `ItemVariationStore`, `vsindex`, and live `blend` evaluation across variation axes.
- **Positioning & Kerning**: Legacy `kern` Format 0 binary search + OpenType `GPOS` `'kern'` LookupType 2 Pair Adjustment (Format 1 individual pairs and Format 2 `ClassDef` pairs) and LookupType 9 Extension subtables.
- **Variable Fonts**: `fvar` axes/instances, `avar` non-linear piecewise segment maps, `ItemVariationStore`, `HVAR` (`DeltaSetIndexMap` advance deltas), and `gvar` packed point/delta streams + phantom advance points.
- **Color Fonts**: `CPAL` multi-palette RGBA tables + `COLR` v0 base/layer records (including `0xFFFF` foreground color entries).
- **Analytic Anti-Aliased Rasterizer**: Adaptive quadratic & cubic curve flattening, exact scanline coverage & area winding accumulation (even-odd or non-zero winding), `to_rgba`, and ASCII preview.

## Module Architecture (`src/`)

| Module | Responsibility |
| --- | --- |
| `binary.ddr` | Big-endian cursor `Reader`, tags, SFNT checksums, UTF-16BE / MacRoman decoders |
| `sfnt.ddr` | SFNT directory, `ttcf` collections, `WOFF1` container & pure-Dunder zlib/DEFLATE Inflate |
| `tables.ddr` | `head`, `maxp`, `hhea`, `hmtx`, `OS/2`, `post`, `name` |
| `cmap.ddr` | `cmap` subtable priority selection and Formats 0/4/6/12/14 lookups |
| `outline.ddr` | FreeType-style vector `Outline`, bounding boxes, affine transforms, adaptive Bezier flattening |
| `glyf.ddr` | `loca` (16/32-bit) and `glyf` simple + composite outline decoding + `gvar` deltas |
| `cff.ddr` | `CFF ` and `CFF2` Type 2 charstring VM, subroutines, `FDSelect`, and `blend` |
| `var.ddr` | `fvar`, `avar`, `ItemVariationStore`, `HVAR`, `gvar` |
| `gpos.ddr` | `kern` Format 0 + `GPOS` PairPos Format 1/2 + LookupType 9 Extension |
| `color.ddr` | `CPAL` + `COLR` v0 layered color lookup |
| `raster.ddr` | Analytic scanline anti-aliased `Bitmap` rasterizer + RGBA & ASCII helpers |
| `font.ddr` | High-level `Face` API (`from_file`, `from_bytes`, metrics, axes, kerning, `render_glyph`, `render_text`) |
| `main.ddr` | `skrift` CLI binary |

## CLI Usage

Build and run the CLI tool:

```sh
dunder build
./build/arm64-macos/dunder-skrift info /System/Library/Fonts/Supplemental/Arial.ttf
./build/arm64-macos/dunder-skrift axes /System/Library/Fonts/NewYork.ttf
./build/arm64-macos/dunder-skrift glyph /System/Library/Fonts/NewYork.ttf g --size 20 --axis wght=700 --axis opsz=14
./build/arm64-macos/dunder-skrift render /System/Library/Fonts/Supplemental/STIXGeneral.otf "AVA" --size 18
./build/arm64-macos/dunder-skrift kern /System/Library/Fonts/Supplemental/Arial.ttf A V
./build/arm64-macos/dunder-skrift info /System/Library/Fonts/Helvetica.ttc --face 1
./build/arm64-macos/dunder-skrift layers <color-font.ttf> <char|U+XXXX>
```

## Library Usage

With `dunder.toml`:

```toml
[dependencies]
skrift = "../dunder-skrift"
```

Then in Dunder:

```dunder
import skrift.font as font

fn main() {
    mut face := font.Face::from_file("/System/Library/Fonts/Supplemental/Arial.ttf", 0)
    if !face.ok { print(face.error); return }
    face.set_pixel_size(24.0)
    rt := face.render_text("AVATAR")
    print(f"bitmap: {rt.width}x{rt.height}, advance={rt.advance_x}")
}
```

## Tests

Run the synthetic + system-font test suite (covers binary readers, SFNT/TTC/WOFF1/WOFF2, stored/fixed/dynamic DEFLATE, `cmap` 0/4/6/12/14, `glyf`, `CFF`, `CFF2`, `kern`/`GPOS`, `fvar`/`avar`/`HVAR`/`gvar`, `CPAL`/`COLR` v0, and rasterization):

```sh
dunder test
```

## Dunder Toolchain Dogfood Findings

Building `dunder-skrift` directly exposed and motivated general improvements in the `dunder` toolchain:

1. **Local path `[dependencies]` in `dunder.toml`**: Enables multi-repo Dunder libraries like `dunder-skrift` to be imported as `import skrift.font` without fragile relative path escaping.
2. **Big-endian primitives in `std.bytes`**: Added `u16_be`, `i16_be`, `u32_be`, `i32_be`, `i64_be`, `u64_be`, and `append_be` symmetric with the existing little-endian APIs.
3. **Observed language ergonomics notes**: Aggregate element arrays (`[]Struct`) are not yet supported (requiring Structure-of-Arrays layouts for tables like `CffTable` and `FvarTable`), and multiline `.method()` chains require outer parentheses.
