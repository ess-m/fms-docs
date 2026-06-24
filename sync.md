# GBA sync protocol

The GBA sync protocol in FMS uses the GBA link port in normal serial mode with a single byte message stream.

## Clock tick

FMS uses a 24PPQN clock and expects ticks at 24 × BPM ÷ 60

## Wiring

The `SO` (serial out), `SI` (serial in), `SC` (serial clock) on the link cable/ext port are used.

Sync OUT:

- FMS `SO` → EXT `SI`
- FMS `SC` → EXT `SC`

Sync IN:

- EXT `SO` → FMS `SI`
- EXT `SC` → FMS `SC`

## Registers

`REG_BASE = 0x04000000`

| Register       | Address      |
|----------------|--------------|
| `REG_RCNT`     | `0x04000134` |
| `REG_SIOCNT`   | `0x04000128` |
| `REG_SIODATA8` | `0x0400012A` |

Send:

```
REG_RCNT   = 0x0000; // normal serial mode
REG_SIOCNT = 0x0001; // 8-bit, internal clock
```

Receive:

```
REG_RCNT   = 0x0000; // normal serial mode
REG_SIOCNT = 0x4080; // 8-bit, external clock, serial IRQ, armed
```

`REG_SIOCNT` bits used: `0x0001` internal clock, `0x0080` start/arm, `0x4000` serial IRQ enable (8-bit Normal mode is `0x0000`)

## Messages

One byte per message in `REG_SIODATA8`:

| Byte   | Msg   | Description                                     |
|--------|-------|-------------------------------------------------|
| `0x01` | TICK  | One clock pulse — 24 per quarter note (24 PPQN) |
| `0x02` | START | Begin playback                                  |
| `0x03` | STOP  | Stop playback                                   |

TICK is only sent/received between START and STOP.

The sequence is the same in both directions: write/read `REG_SIODATA8` and set `REG_SIOCNT |= 0x0080` to send the message, or to arm for the next one when receiving.