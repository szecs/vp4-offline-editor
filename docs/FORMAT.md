# VP4 file format notes

Worked out from VP4 backup files (firmware 1.x–4.x). Not official documentation: use at your own risk.

## SysEx framing

Every message: `F0 00 01 74 14 <fn> … <cks> F7`

- `14` = VP4 model id (FM3 uses `11`, with the same scheme).
- `cks` = XOR of all preceding bytes (including `F0`), `& 0x7F`.
- 16-bit values are packed into 3 bytes: `v = b0 | b1 << 7 | b2 << 14` (7 + 7 + 2 bits).

## Preset file

| fn | Message | Content |
|----|---------|---------|
| `77` | header | bank (0–25 = A–Z), slot (0–3), word count (3 bytes, `0x1000`) |
| `78` | data ×4 | 2-byte word count (`0x400`), then 1024 packed words |
| `79` | footer | 3-byte value = XOR of all 4096 data words |

An "allBanks" file is simply 104 presets back to back.

### Data words

| Word | Meaning |
|------|---------|
| 0 | format version |
| 1 | `0xAA55` |
| 2 | CRC-16/CCITT (poly `0x1021`, init `0xAA55`) over all 4096 words, each low byte first, with word 2 taken as 0 |
| 4–19 | preset name: 32 chars, 2 per word, low byte first, space padded, last char NUL |
| 36 | unpacked body size (bytes) |
| 37 | compressed body size (bytes) |
| 38… | compressed body; rest of the 4096 words is zero |

The bank/slot in the `77` header is not covered by the CRC, so moving a preset only changes that message.

### Compressed body

A bitstream: bytes taken low byte first from the words, bits read MSB-first.

1. A pre-order Huffman tree: bit `0` = branch (left subtree, then right), bit `1` = leaf followed by its 8-bit byte value.
2. The Huffman-coded bytes (`0` = left, `1` = right), until the unpacked size is reached.

Fractal's encoder can be reproduced bit for bit: leaves in ascending byte order; repeatedly take the two lowest weights (ties: newest node first) and make the first one the left child; codes are at most 13 bits.

### Unpacked body (16-bit little-endian words)

- **Scenes**: 4 records of 40 words. Name at word +2 (16 words). Word +18+2i = channel (0–3) of slot i, word +26+2i = 1 if slot i is bypassed in that scene.
- **Blocks**, from about word 160: `[blockId, 0, 0, 0, 0, paramCount, channelCount]` followed by `paramCount × channelCount` values (channel A first). Block ids follow Fractal's Axe-Fx III numbering minus one, e.g. 37 Input, 42 Output, 46 Compressor, 66 Reverb, 70 Delay, 90 Phaser, 94 Wah, 118 Drive, 178 Plex Delay, 206 Routing; 2 = Controllers, 3–18 = Modifiers.

### Parameter values

- Continuous: `0…65534` mapped to min…max, linearly or logarithmically depending on the parameter type.
- Lists and integers: stored directly. Lists whose range starts at 1 (e.g. MIDI CC lists) are stored as index + 1.
- Common block parameters: 0 Level, 1 Mix, 2 Balance, 3 Bypass, 4 Bypass Mode, 6 Scene Ignore, 10 Type.
- Preset tempo: Controllers block, parameter 28, in BPM.

### Routing block (id 206)

| Param | Meaning |
|-------|---------|
| 15–18 | block id in slots 1–4 (0 = empty) |
| 20–22 | Series/Parallel for FX2–FX4 (FX1 and the first POST effect are always series) |
| 23+4s+i | channel of slot i in scene s (same as the scene record) |
| 39+4s+i | slot i bypassed in scene s (same as the scene record) |
| 63 | 4CM mode: 0 disabled, n = (n−1) pre / rest post |
| 64–111 | scene MIDI: message, channel, value, 4 per scene |

Per-scene channel and bypass are stored twice (routing block and scene records); both copies must be kept in sync.

## System file

| fn | Message | Content |
|----|---------|---------|
| `51` | header | word count (2 bytes; 164 with firmware 4.x) |
| `52` | data | 2-byte word count, then packed words |
| `53` | footer | 3-byte value = XOR of all words |

Word index = Global parameter id (e.g. 13 = tuner calibration 430–450 Hz, 15 = spillover, 47 = analog input pad, 74 = scene select CC).
