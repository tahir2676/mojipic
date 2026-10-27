# Mojipic BLE protocol

Reverse-engineered from the Mojipic Android app v4.8.0 (`com.mojipic.mojipic2`),
module `com.mojipic.tools_bluetooth_kit`. Everything below runs over local
Bluetooth Low Energy — **no server is involved** in controlling the display.

## Discovery

* Advertised name: `Mojipic` (the app scans with `ScanFilter.setDeviceName("Mojipic")`).
* Main service: `0xFFF0`. Battery service: `0x180F` (char `0x2A19`).
* The app matches characteristics by the 16-bit short UUID (chars 5–8 of the
  full UUID), across all services.

| Short UUID | App name | Use |
|---|---|---|
| `FFF1` | WRITE_IMAGE | image upload packets (20 B) |
| `FFF2` | STATUS_REQUEST | **notify** – status byte (flow control) |
| `FFF3` | IMAGE_ID_TO_DISPLAY | show stored image by id (2 B LE), notify = current id |
| `FFF4` | READ_IMAGES_COUNT | number of stored images |
| `FFF5` | READ_IMAGES_IDS | id readback (write `A5`, notify, read 4 B) |
| `FFF6` | DEVICE_COMMANDS | control commands (see below) |
| `FFF7` | ERASE_IMAGE_BY_ID | erase one image (2 B LE id) |
| `FFF8` | TIME_AFTER_DISCONNECTING | 1 B: auto-off seconds after disconnect, 1–180 (0 = never) |
| `FFF9` | BRIGHTNESS_SETTING | 1 B, 1–100 |
| `FFFB` | SINGLE_COLOR_PIXEL_RECORDING | live drawing (18 B) |
| `FFFC` | RECORD_PIXELS_TWO_COLOR | Tetris game pixels |
| `FFFD` | UNIQUE_PROCESSOR_ID | read – used for handshake |
| `FFE0` | CREEPING_LINE_RECORDING | scrolling text (20 B packets) |
| `FFE1` | BLUETOOTH_POWER | 1 B (0 = low TX power, app writes 1) |
| `FFE2` | CHECK_SERIAL_NUMBER | handshake write + notify |
| `2A19` | BATTERY_LEVEL | read, percent |
| `2A26` | FIRMWARE_VERSION | read |

## Connection handshake (required)

1. Read `FFFD` → processor id `sn` (≥ 12 bytes).
2. Enable notifications on `FFE2`.
3. Write 20 bytes to `FFE2`: random bytes, except
   `[5]=sn[3] [7]=sn[10] [8]=sn[1] [11]=sn[7] [14]=sn[4] [18]=sn[9]`.
4. Device notifies on `FFE2`. App validates
   `h[2]==sn[11] h[6]==sn[1] h[7]==sn[5] h[13]==sn[1] h[16]==sn[8] h[17]==sn[3]`.
5. Turn on: `FFF6 ← 02 01`, then screen mode `FFF6 ← 02 06 00`.

(The app also reads `FFE1`; if it is `0` it does screen-off, writes `FFE1 ← 01`,
turns the device off and waits 900 ms. This only raises TX power and is optional.)

## Control commands (`FFF6`)

| Bytes | Meaning |
|---|---|
| `02 01` | device on |
| `02 02` | device off |
| `02 06 05` | screen on |
| `02 05` | screen off |
| `07 01` | clear screen |
| `07 00` | show last picture |
| `02 06 00` | screen mode: constant |
| `07 02 ss` | screen mode: slideshow, `ss` seconds per picture |
| `02 06 ss` | screen mode: timed, `ss` seconds |
| `EA E0` | **erase all pictures** (destructive) |

## Images

* 32 × 32 pixels, 3 bytes per pixel (R, G, B), rows top-to-bottom,
  **each channel 5-bit (0–31)**. Frame = 3072 bytes. Up to 20 frames (app limit).
* Device id (16 bit) = `(type << 12) | id` (id 12 bit):
  * type `2` = still full-colour, `3` = animated full-colour (channel = `c >> 3`)
  * type `0` = still 1-bit, `1` = animated 1-bit (channel = `c > 127 ? 31 : 0`);
    used by built-in library items with id > 237 whose name lacks "(8bit)".
* Built-in library ids: 1–1997. User uploads in this tool default to 3000+.

### Upload (`FFF1`)

For each frame `f`, for each packet `p` in 0..191:

```
[ id>>8 , id&255 , p , f , data[f*3072 + p*16 + 0..15] ]   (20 bytes)
```

Before each packet wait for a `FFF2` notification whose status byte has
bit `0x01` (WRITE_BLOCK_SUCCESS) or `0x04` (FLASH_READY). The first packet is
sent immediately. Other status bits: `0x02` block error, `0x08` flash not ready,
`0x20` image-id ready, `0x40` image written, `0x10` slave MCU ready, `0x80` MCU on.

Then show it: `FFF3 ← id&255, id>>8` (write without response).

## Live drawing (`FFFB`)

18 bytes: `[R, G, B, x1, y1, x2, y2, …]`, colour channels `0` or `31`
(app thresholds each channel at > 16), up to 7 pixels, unused bytes `FF`.
The app clears the screen (`07 01`) when entering live mode.

## Scrolling text (`FFE0`)

Characters are mapped to the device font with a 290-entry table
(`TickerCalculator.getTickerCharList`; unknown → `0` = space). For
`k = 0 .. floor(n/8)` send:

```
[ n , 0 , delay , (size<<3)|color , c[8k] hi, lo, … c[8k+7] hi, lo ]   (20 bytes, big-endian codes, 0 past end)
```

* `delay` 3–60 (lower = faster; app: `60 − slider% × 57/100`)
* `size` 0–2 (app sizes 1–3 minus 1)
* `color` 0 black, 1 white, 2 red, 3 green, 4 blue, 5 yellow, 6 pink, 7 sky

## Stored-id readback (`FFF5`)

Enable notify on `FFF5`; read `FFF5` (id = `(b[2]<<8 | b[3]) & 4095`),
write `A5`, wait for notify, read again… repeat for the stored count.
