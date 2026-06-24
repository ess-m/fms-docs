# ROM data

| Data                  | Offset    | Bytes | Format          | Symbol             |
|-----------------------|-----------|-------|-----------------|--------------------|
| Phase increment table | `0x1713C` | 480   | 120 × u32       | `phaseIncLUT`      |
| Theme palettes        | `0x1A378` | 16    | 2 × (4 × u16)   | `themes`           |
| Accent colors (dark)  | `0x1A388` | 16    | 8 × u16         | `accentsDark`      |
| Accent colors (light) | `0x1A398` | 16    | 8 × u16         | `accentsLight`     |
| Font glyphs           | `0x1A770` | 320   | 80 × u32        | `FONT_DATA`        |
| Env curve table       | `0x1C928` | 2048  | 1024 × u16      | `curveLUT`         |
| Sine table            | `0x1D128` | 4096  | 2048 × s16      | `sinLUT`           |

## Colors

Colors are in BGR555 format: a `u16` with red in bits 0–4, green in bits 5–9, blue in bits 10–14; each channel 0–31.

**Theme**

| Color            | Dark      | Light     |
|------------------|-----------|-----------|
| Background       | `0x1A378` | `0x1A380` |
| Dim              | `0x1A37A` | `0x1A382` |
| Bright           | `0x1A37C` | `0x1A384` |
| Accent (default) | `0x1A37E` | `0x1A386` |

**Accent** - overrides the theme's default (red) accent

| Color  | Dark      | Light     |
|--------|-----------|-----------|
| Red    | `0x1A388` | `0x1A398` |
| Green  | `0x1A38A` | `0x1A39A` |
| Blue   | `0x1A38C` | `0x1A39C` |
| Yellow | `0x1A38E` | `0x1A39E` |
| Pink   | `0x1A390` | `0x1A3A0` |
| Orange | `0x1A392` | `0x1A3A2` |
| Purple | `0x1A394` | `0x1A3A4` |
| Teal   | `0x1A396` | `0x1A3A6` |

## Font

Each glyph is a `u32` holding a 5×5 1-bpp bitmap (source: font.png)
Bit `n` is the pixel at row `n / 5`, column `n % 5` — bit 0 = top-left, bit 24 = bottom-right; a set bit is foreground.

## LUTs

- **Phase increment table** — one `u32` per note, index 0 = C0 .. 119 = B9. 

  ```
  freq(n) = 440 * 2^((n - 57) / 12) // 12-TET tuning
  phaseInc(n) = round(freq(n) * 2^32 / 32768) // samplerate = 32768
  ```

- **Sine table** - one half-period of a sinewave, `sinLUT[i] = round(32767 * sin(π * i / 2048))`
(The engine sign-folds it for the negative half)

- **Curve table** - amplitude curve for the FM envelopes. `curveLUT[0] = 0`, `curveLUT[1023] = 65535`.