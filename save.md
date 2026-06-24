# Save format

v 20

## Storage layout

| Build  |                                  | Size  | Sectors           | ROM tag        |
|--------|----------------------------------|-------|-------------------|----------------|
| Flash  | GBA 128 KB Flash, two 64 KB banks| 128 KB| 32 × 4 KB         | `FLASH1M_V103` |
| SRAM   | GBA 32 KB SRAM                   | 32 KB | 8 × 4 KB          | `SRAM_V113`    |

- Sector size: `0x1000` bytes (4 KB)
- Sector 0 holds the directory + global settings.
- Sectors 1..N hold the bank data pool (`DATA_SECTORS` = 31 on Flash, 7 on SRAM).
- All multi-byte values are little-endian.

## Sector 0 – directory + global settings (4 KB)

| Offset  | Size | Field            | Notes                                                              |
|---------|------|------------------|--------------------------------------------------------------------|
| 0x0000  |   4  | magic            | ASCII `GSEQ`                                                       |
| 0x0004  |   2  | version          | Save format version (u16). Current = 20                            |
| 0x0006  |   2  | bpm              | Tempo, 30–300 (u16)                                                |
| 0x0008  |   2  | bankValid        | Bitmask of occupied banks (u16). Bit N = bank N occupied           |
| 0x000A  |   2  | scaleMask        | 12-bit scale bitmask (u16, low 12 bits)                            |
| 0x000C  |   1  | accentColor      | UI accent color index (u8)                                         |
| 0x000D  |   1  | syncMode         | 0 = off, 1 = out, 2 = in (u8)                                      |
| 0x000E  |   1  | syncFormat       | 0 = GBA, 1 = analog, 2 = ardboy, 3 = pmidi, 4 = audio (u8)         |
| 0x000F  |   1  | theme            | Color theme index (u8)                                             |
| 0x0010  |  16  | bankDir[8]       | Bank directory: 8 entries of `{ u8 sectorStart, u8 sectorCount }`  |
| 0x0020  |   1  | dpadSwap         | D-pad orientation swap (u8), 0 or 1                                |
| 0x0021  |   1  | abSwap           | A/B button swap (u8), 0 or 1                                       |
| 0x0022  |   1  | bankLocked       | Per-bank lock bitmask (u8). Bit N = bank N locked                  |
| 0x0023  |  32  | bankNames[8]     | 8 banks × 4 chars (u8).                                            |
| 0x0043  |   1  | activeBank       | Last active bank index 0–7 (u8)                                    |
| 0x0044  |  16  | bankBPM[8]       | Per-bank tempo override (u16 each, 0 = unset)                      |
| 0x0064  |  16  | bankScale[8]     | Per-bank scale override (u16 each, 0 = unset)                      |
| 0x0084  |   1  | syncPpqDivider   | Clock/audio sync PPQ divider (u8)                                  |
| 0x0085  |  16  | fmPresetValid[16]| FM preset marker per slot (u8). `0xA5` = stored, else empty        |
| 0x0095  |  16  | nsPresetValid[16]| Noise preset marker per slot (u8). `0xA5` = stored, else empty     |
| 0x00A5  | 210  | fmPresets[15]    | `FMPreset` (14 B) for slots 1–15.                                  |
| 0x0177  |  90  | nsPresets[15]    | `NoisePreset` (6 B) for slots 1–15.                                |
| 0x01D1  |  …   | (unused)         | Padded to end of sector                                            |

### bankDir entries

Each entry is two bytes: `{ sectorStart: u8, sectorCount: u8 }`.

- `sectorStart` is the first sector (1–31 on Flash, 1–7 on SRAM) allocated to that bank.
- `sectorCount` is the number of contiguous sectors the bank occupies.
- `sectorStart == 0` means the bank has never been saved.
- Banks are packed in index order; a full rewrite repacks all banks from sector 1 onward.

### Bank name encoding

| Value       | Display               |
|-------------|-----------------------|
| 0–9         | `'0'`–`'9'`           |
| 10–35       | `'A'`–`'Z'`           |
| 36          | `'-'`                 |
| 0xFF        | `'-'` (uninitialized) |

### Sound presets

There are 16 slots. Slot 0 is the default (init) sound and is never stored; slots 1–15 are stored.
A slot counts as stored when its marker byte (`fmPresetValid` / `nsPresetValid`) equals `0xA5`.

Markers are slot-indexed, preset bodies are `(slot - 1)`-indexed:

- marker at `0x0085 + slot` (FM) / `0x0095 + slot` (noise)
- body at `0x00A5 + (slot - 1) × 14` (FM) / `0x0177 + (slot - 1) × 6` (noise)

`FMPreset` (14 bytes)

| Offset | Type | Field        |
|--------|------|--------------|
|   0    | u8   | level        |
|   1    | u8   | porta        |
|   2    | u8   | modMult      |
|   3    | u8   | modLevel     |
|   4    | u8   | modFbk       |
|   5    | u8   | modRelease   |
|   6    | u8   | modAttack    |
|   7    | u8   | modEnd       |
|   8    | u8   | attack       |
|   9    | u8   | hold         |
|  10    | u8   | release      |
|  11    | u8   | sweepRelease |
|  12    | u8   | sweepLevel   |
|  13    | u8   | oscMode      |

`NoisePreset` (6 bytes)

| Offset | Type | Field  |
|--------|------|--------|
|   0    | u8   | rate   |
|   1    | u8   | level  |
|   2    | u8   | attack |
|   3    | u8   | hold   |
|   4    | u8   | decay  |
|   5    | u8   | porta  |

## Sectors 1..N – bank data pool

Each bank is stored as a contiguous run of 1–5 sectors.
A bank consists of a fixed 0x0A62-byte metadata region followed by variable-length compressed pattern data.

### Fixed metadata region (0x0A62 bytes)

| Offset  | Size | Field                      | Notes                                                |
|---------|------|----------------------------|------------------------------------------------------|
| 0x0000  |  15  | track metadata             | See "Track metadata layout" below                    |
| 0x000F  |   1  | bankVersion                | Version + bank-id stamp.                             |
| 0x0010  |  80  | slotSaved[5][16]           | Which patterns have data                             |
| 0x0060  |  64  | fmPatLength[4][16]         | Pattern lengths per FM track/slot (u8)               |
| 0x00A0  |  64  | fmPatRate[4][16]           | Rate dividers per FM track/slot (u8)                 |
| 0x00E0  |  16  | nsPatLength[16]            | Pattern lengths per noise slot (u8)                  |
| 0x00F0  |  16  | nsPatRate[16]              | Rate dividers per noise slot (u8)                    |
| 0x0100  |  64  | fmEchoRepeats[4][16]       | Echo repeat count (u8)                               |
| 0x0140  |  64  | fmEchoInterval[4][16]      | Echo interval in PPQ ticks (u8)                      |
| 0x0180  |  64  | fmEchoStereo[4][16]        | Echo stereo mode (u8)                                |
| 0x01C0  |  64  | fmEchoVolDecay[4][16]      | Echo volume decay (s8)                               |
| 0x0200  |  64  | fmEchoModDecay[4][16]      | Echo mod decay (s8)                                  |
| 0x0240  |  64  | fmEchoTranspose[4][16]     | Echo transpose, semitones (s8, –24 .. +24)           |
| 0x0280  |  64  | fmEchoTspAccum[4][16]      | Echo transpose accumulate flag (u8, 0 or 1)          |
| 0x02C0  |  64  | fmShuffle[4][16]           | FM shuffle amount per track/slot (u8, 0–15)          |
| 0x0300  |  16  | nsShuffle[16]              | Noise shuffle amount per slot (u8, 0–15)             |
| 0x0310  |  16  | nsEchoRepeats[16]          | Noise echo repeat count (u8)                         |
| 0x0320  |  16  | nsEchoInterval[16]         | Noise echo interval in PPQ ticks (u8)                |
| 0x0330  |  16  | nsEchoStereo[16]           | Noise echo stereo mode (u8)                          |
| 0x0340  |  16  | nsEchoVolDecay[16]         | Noise echo volume decay (s8)                         |
| 0x0350  |  16  | nsEchoRateShift[16]        | Noise echo rate shift (s8)                           |
| 0x0360  |  16  | nsEchoRateAccum[16]        | Noise echo rate accumulate flag (u8)                 |
| 0x0370  | 512  | fmTransposeRates[4][16][8] | Pattern loops per transpose-step (u8, 1–15)          |
| 0x0570  |  64  | fmTransposeLen[4][16]      | Transpose sequencer list length (u8, 1–8)            |
| 0x05B0  | 512  | fmTransposeSteps[4][16][8] | Semitone offsets per step (s8, –24 .. +24)           |
| 0x07B0  |  64  | fmTransposeMode[4][16]     | Advance mode per pattern (u8, 0 = PTN, 1 = STEP, 2 = TRIG) |
| 0x07F0  | 384  | fmPatMods[4][16]           | Mod config per FM track/slot (`ModConfig`, 6 B each) |
| 0x0970  |  96  | nsPatMods[16]              | Mod config per noise slot (`ModConfig`, 6 B each)    |
| 0x09D0  |   2  | extVersion                 | Real per-bank version (u16).                         |
| 0x09D2  |  64  | fmDirection[4][16]         | FM playback direction (u8, 0 = FWD, 1 = PNP, 2 = BWD, 3 = RND) |
| 0x0A12  |  16  | nsDirection[16]            | Noise playback direction (u8, 0 = FWD, 1 = PNP, 2 = BWD, 3 = RND) |
| 0x0A22  |  64  | fmEchoFbkDecay[4][16]      | FM feedback echo decay (s8)                          |
| 0x0A62  |  —   | (end of metadata)          | Start of compressed pattern stream                   |

### Track metadata layout (0x0000–0x000E, 15 bytes)

Interleaved by field, then by track. Tracks 0–3 are FM, track 4 is noise.

| Bytes      | Field                                              |
|------------|----------------------------------------------------|
| `[0..3]`   | FM tracks 0–3 current pattern index (u8 each)      |
| `[4]`      | Noise track current pattern index (u8)             |
| `[5..8]`   | FM tracks 0–3 length (u8 each)                     |
| `[9]`      | Noise track length (u8)                            |
| `[10..13]` | FM tracks 0–3 rateDiv (u8 each)                    |
| `[14]`     | Noise track rateDiv (u8)                           |

### bankVersion byte (offset 0x000F)

Current banks store a sentinel here rather than the version directly:

- High nibble: bank-id stamp — the bank index this bank claims to be (0–7).
- Low nibble: `0x1` sentinel, meaning the real per-bank version is the u16 `extVersion` at offset 0x09D0.

A fully-erased sector reads as `0xFF`.
Banks written by older firmware encode the version in the low nibble.

### ModConfig layout (6 bytes)

| Offset | Type | Field  | Notes                                              |
|--------|------|--------|----------------------------------------------------|
|   0    | u8   | target | Modulation target track, 0–4                       |
|   1    | u8   | dst    | Destination parameter, 0–12                        |
|   2    | u8   | speed  | Cycle length in triggers, 1–255                    |
|   3    | u8   | shape  | LFO shape (see ModShape), 0–4                      |
|   4    | s8   | depth  | Amplitude endpoint, ±127 (0 = inactive)            |
|   5    | u8   | offset | LFO phase offset, 0–15                             |

**ModDst** (destination parameter): 0=LEVEL, 1=MODMULT, 2=MODLEVEL, 3=MODFBK,
4=MODATK, 5=MODREL, 6=ATK, 7=HLD, 8=REL, 9=N_RATE, 10=N_ATK, 11=N_HLD, 12=N_DEC.
Values 0–8 target an FM parameter; 9–12 target the noise track.

**ModShape** (LFO shape): 0=DOWN, 1=UP, 2=UP_DOWN, 3=SQUARE, 4=RANDOM.

### Compressed pattern data (variable length, starts at offset 0x0A62)

Patterns are stored sequentially: FM tracks 0 to 3, then noise (4)
Only patterns where `slotSaved[track][pattern] == 1` are present.

Each saved pattern is encoded as:

```
u16 trigMask // bitmask of active steps (bit N = step N has data)

for each set bit N in trigMask (step 0..15):
    u8 trig // trig value: 1 = TRIG_ON, 2 = TRIG_LESS
    u8 data[stepSize - 1] // remaining step data (excluding the trig byte)
```

- `stepSize` = 24 for FM tracks (FMStep), 12 for noise (NoiseStep).
- The trig byte is written first, then the remainder of the step struct (offsets 1 .. stepSize-1) is copied verbatim.

### FMStep layout (24 bytes)

| Offset | Type | Field        | Notes                                          |
|--------|------|--------------|------------------------------------------------|
|   0    | u8   | trig         | 0 = off, 1 = on, 2 = less                      |
|   1    | u8   | note         | MIDI note number, 0–119                        |
|   2    | u8   | level        | Carrier volume, 0–127                          |
|   3    | u8   | pan          | Stereo panning, 0–2 (0=L, 1=C, 2=R)            |
|   4    | u8   | count        | Modulo count                                   |
|   5    | s8   | mask         | Modulo bitmask                                 |
|   6    | u8   | modMult      | Modulator frequency multiplier index, 0–127    |
|   7    | u8   | modLevel     | Modulator level, 0–127                         |
|   8    | u8   | modFbk       | Modulator feedback, 0–127                      |
|   9    | u8   | modRelease   | Modulator release, 0–127                       |
|  10    | u8   | modAttack    | Modulator attack, 0–127                        |
|  11    | u8   | attack       | Carrier attack, 0–127                          |
|  12    | u8   | hold         | Carrier hold, 0–127                            |
|  13    | u8   | release      | Carrier release, 0–127                         |
|  14    | u8   | sweepRelease | Pitch sweep release, 0–127                     |
|  15    | u8   | sweepLevel   | Pitch sweep level, 0–127                       |
|  16    | u8   | porta        | Portamento, 0–127                              |
|  17    | u8   | modEnd       | Modulator-envelope sustain floor, 0–127        |
|  18    | u16  | chordPacked  | 3 × 4-bit chord intervals (see below)          |
|  20    | u8   | echoSend     | 0 = send to echo, 1 = muted                    |
|  21    | u8   | oscMode      | 0 = FM, 1 = additive                           |
|  22    | u8   | tspGate      | 0 = transposed, 1 = not transposed             |
|  23    | u8   | wait         | Step offset in sixths of a step, 0–6           |

**chordPacked** — a `u16` packing three 4-bit chord intervals:

- slot 0 = bits `[3:0]`, slot 1 = bits `[7:4]`, slot 2 = bits `[11:8]`
- each nibble is a semitone interval above the step's `note`
- extract: `(chordPacked >> (slot * 4)) & 0xF`

### NoiseStep layout (12 bytes)

| Offset | Type | Field    | Notes                                              |
|--------|------|----------|----------------------------------------------------|
|   0    | u8   | trig     | 0=off, 1=on, 2=less                                |
|   1    | u8   | rate     | Noise rate, 0–112                                  |
|   2    | u8   | level    | Volume, 0–15                                       |
|   3    | u8   | pan      | Stereo panning, 0–2 (0=L, 1=C, 2=R)                |
|   4    | u8   | count    | Modulo count                                       |
|   5    | s8   | mask     | Modulo bitmask                                     |
|   6    | u8   | attack   | Attack, 0–7                                        |
|   7    | u8   | hold     | Hold, 0–63                                         |
|   8    | u8   | decay    | Decay, 0–21                                        |
|   9    | u8   | porta    | Portamento, 0–127                                  |
|  10    | u8   | echoSend | 0 = send to echo, 1 = muted                        |
|  11    | u8   | wait     | Step offset in sixths of a step, 0–6               |

## Versioning

There are two version numbers:

- **Directory version** (sector 0, offset 0x0004) — the format the file was last written with.
  
- **Per-bank version** (bank offset 0x000F / `extVersion` at 0x09D0) 
Banks upgrade lazily, keeping their previous layout until the next time that bank is saved.

## Capacity

- `META_SIZE` (per-bank metadata) = `0x0A62` bytes.
- Per-bank maximum on-flash size = 5 sectors × 4 KB = 20 KB. Save attempts that would exceed this are rejected.
- Total data pool = 31 sectors (Flash) / 7 sectors (SRAM). The 8 banks share this pool.
