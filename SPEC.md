# Clean Flash image format, draft 2

A Clean Flash image (`.cfw`) is a 4 KiB header followed by flash sectors,
exactly as they are stored in flash. It describes part of a unit's flash: the
firmware, the bootloader, or both. A writer installs it so that every sector in
that part ends up exactly as the image says, apart from the user's data on an
upgrade the user chose to keep.

The words MUST, MUST NOT, SHOULD and MAY are used as in RFC 2119.

## 1. Units and terms

| Board | Units | Flash |
|---|---|---|
| 1, FM-1 | M-VAVE FM-1, Fame MiniFM | 1 MiB, 256 sectors of 4 KiB |
| 2, SMK-37 | M-VAVE SMK-37 Pro, SMK-37 Elite | 1 MiB, 256 sectors of 4 KiB |

- **Sector:** 4 KiB of flash. Sector *n* is `n × 0x1000` to `n × 0x1000 + 0xFFF`.
- **Boot area:** sectors 0 to 3 (`0x00000`–`0x03FFF`): the flash header, the
  boot file table, the bootloader and its configuration.
- **Flash chip:** the 1 MiB SPI NOR flash, a die inside the JieLi chip's
  package, from more than one maker. Its JEDEC ID (command `9F`) names the
  die's maker and size, not the board.
- **Chip key:** the 16-bit key the chip decrypts the app area (from `0x4000`)
  with. Firmware is stored encrypted with it. Every unit and every package seen
  so far uses `0x980F`.
- **Writer:** whatever installs an image: M-Flasher in download mode, a web
  page through m-bootloader's MIDI mode or RAM stage, a program on the device.
- **Packager:** whoever makes an image: a firmware's author, or anyone
  repackaging an open-source firmware.

## 2. Scopes and protected ranges

An image has one **scope**, which fixes the sectors it writes. A writer writes
every sector in the scope and no other. No sector in the scope is left as it
was, except the user's data kept on an upgrade (section 4.2).

| Scope | FM-1 | SMK-37 |
|---|---|---|
| 1, firmware | `0x04000`–`0xFEFFF` | `0x04000`–`0xBFFFF`, `0xC2000`–`0xFEFFF` |
| 2, bootloader | `0x00000`–`0x03FFF` | `0x00000`–`0x03FFF` |
| 3, bootloader and firmware | both of the above | both of the above |

**Protected ranges** are in no scope. No writer writes them:

| Board | Range | Holds |
|---|---|---|
| both | `0xFF000`–`0xFFFFF` | key area (`key_mac`): factory Bluetooth address and flash efuse copy where present; m-bootloader's calibration copy (`TRIMCOPY`) on the SMK-37 |
| SMK-37 | `0xC0000`–`0xC0FFF` | BTIF: the Bluetooth address and pairings, in the JieLi SDK's format |
| SMK-37 | `0xC1000`–`0xC1FFF` | USRTRIM: the factory key, pad and wheel calibration (`trimdat`) |

A range is protected only if every known firmware either never writes it or
uses it in one shared format. New scopes or protected ranges need a new format
version.

## 3. The file

| Offset | Size | Contents |
|---:|---:|---|
| 0 | 4096 | Header |
| 4096 | 4096 × *k* | The stored sectors, in address order |

The file is exactly `4096 × (1 + k)` bytes, where *k* is the number of bits set
in the stored map.

### 3.1 Header

All integers are little endian. Every reserved byte is 0.

| Offset | Size | Field |
|---:|---:|---|
| 0 | 4 | Magic: `43 46 57 1A` (`CFW`, then `0x1A`) |
| 4 | 2 | Format version: 1 |
| 6 | 1 | Board (section 1) |
| 7 | 1 | Scope (section 2) |
| 8 | 4 | Reserved |
| 12 | 4 | CRC-32 of the stored sectors (all bytes from 4096 to the end) |
| 16 | 32 | SHA-256 of the stored sectors |
| 48 | 32 | Name, UTF-8, padded with zeros, for example `Melodee` |
| 80 | 16 | Version, UTF-8, padded with zeros, for example `1.6.1` |
| 96 | 32 | Stored map |
| 128 | 32 | User map |
| 160 | 32 | Flash chips: the JEDEC IDs the firmware accepts, up to eight, 4 bytes each; unused ones 0 |
| 192 | 2 | Number of predecessors, 0 to 32 |
| 194 | 2 | Reserved |
| 196 | 384 | Predecessors: 32 entries of 12 bytes; unused ones 0 |
| 580 | 3512 | Reserved |
| 4092 | 4 | CRC-32 of bytes 0 to 4091 |

CRC-32 is the IEEE polynomial as in zlib (reflected, initial and final value
`0xFFFFFFFF`), the check the chip computes on a flash range.

**Maps** are 32 bytes, one bit per sector: bit *n* is bit *n* mod 8 of byte
*n* / 8.

- **Stored map:** bit *n* set means sector *n*'s 4096 bytes are in the file,
  at offset `4096 × (1 + the number of bits set below n)`. In the scope, a clear
  bit means the sector is erased (all `0xFF`). Outside the scope every bit MUST
  be clear.
- **User map:** bit *n* set means sector *n* holds the user's data (settings,
  projects, presets, samples, pairings). The image holds its default content
  for those sectors, or leaves them erased. Bits MAY be set only in the
  firmware scope.

**Flash chips:** the flash parts the image's firmware works on, by JEDEC ID
(command `9F`, as `0x856014`). Firmwares that drive the flash themselves
accept only the parts they know, and on any other part they run without
saving anything and refuse their own updates, so installing one there leaves a
unit that only the chip's ROM download mode can recover (docs/findings.md).
Scopes 1 and 3 MUST list at least one ID. Scope 2 MUST list none: bootloaders
take the flash's size from the ID and work on every part seen.

**Predecessor entry:** an earlier build this image upgrades, found by a
fingerprint of its code:

| Offset | Size | Field |
|---:|---:|---|
| 0 | 4 | Start: a flash address |
| 4 | 4 | Length, more than 0 |
| 8 | 4 | CRC-32 of that range as stored in flash |

The range SHOULD cover only that build's code, not its data, so that the same
build matches whatever the user stored. All predecessors share the user map: an
earlier build whose data lives elsewhere is not listed, and upgrading from it
is a clean install.

### 3.2 Stored sectors

Each stored sector is 4096 bytes, exactly as it is to be in flash: the app area
encrypted with the chip key, as in a `.fwsc` package's `flash.bin`. A packager
SHOULD NOT store a sector that is all `0xFF`; its bit is left clear instead.

## 4. Installing

### 4.1 Checks before anything is written

A writer MUST check all of these, and MUST refuse the image with nothing
written if any fails:

1. The magic, the format version, the header's CRC-32, and the board matching
   the unit.
2. The file size, and the CRC-32 of the stored sectors. A host also checks the
   SHA-256.
3. No stored-map bit outside the scope, no user-map bit outside the firmware
   scope, at most 32 predecessors, flash chips as section 3.1 says for the
   scope.
4. Scopes 1 and 3: the unit's JEDEC ID is one of the flash chips. M-Flasher
   reads it in download mode, a web page from m-bootloader's MIDI mode, a
   program on the device from the flash itself.
5. Scopes 1 and 3: sector `0x4000` is stored, and its first 32 bytes decrypt
   with the unit's chip key to a valid record (their CRC-16 over bytes 2–31
   equals bytes 0–1), as m-bootloader's MIDI mode checks a package.
6. Scopes 2 and 3: the boot area is a bootloader the writer recognises for this
   board, and its chip-key record decodes to the unit's chip key.

A writer that can't hold the image in memory checks the CRC-32 in a first pass
over the file, and writes in a second.

### 4.2 Clean install or keep

For scopes 1 and 3, the writer compares each predecessor's range in the unit's
flash with its CRC-32. The chip can compute it (m-bootloader's MIDI command
`0x24`, M-Flasher's loader).

- **A predecessor matches:** the writer asks whether to keep the user's data.
  Keeping is the default. If kept, the sectors in the user map are left as they
  are, and every other sector in the scope is written.
- **None matches:** a clean install. Every sector in the scope is written. The
  writer says what that means (section 4.5) first.

The new firmware converts kept data on its first start, as firmwares already do
across their own updates.

### 4.3 Writing

- A host SHOULD save a backup of the whole flash before the first write.
- A sector whose flash already holds the bytes to be written is skipped. The
  chip's CRC-32 of the sector is enough to tell.
- Every written sector is verified: read back, or the chip's CRC-32 of it.
- **Scopes 2 and 3: the boot area first,** sectors 3, 2 and 1, then sector 0
  last, so that a boot area cut short has no valid file table.
- **Scopes 1 and 3: the firmware,** with `0x4000` erased first and written
  last, so that until the end nothing in flash can pass as firmware.
- A writer MUST NOT write a protected range, and SHOULD check that again where
  it erases.

### 4.4 Routes

| Writer | Firmware scope | Boot area |
|---|---|---|
| M-Flasher, download mode | writes the plan itself | yes, after a backup and its own confirmation |
| Web page, m-bootloader MIDI mode or RAM stage | serves an ordinary package built from the planned flash: MIDI mode writes `0x4000` up to the package's end, skipping sectors that already match. For kept sectors the page first reads them from the unit (`0x23`) and serves them back | no |
| The device: a RAM app reading the image from an SD card (m-bootloader's SD card item) | yes, in two passes: check, then write | as M-Flasher, if the app implements it |

### 4.5 What the writer tells the user

Before a clean install:

- The data of the firmware installed now (settings, projects, presets,
  samples) is erased. A backup brings it back.
- Bluetooth pairings are erased. On an FM-1 the unit also gets a new Bluetooth
  address, so computers and phones see a new device.
- On an SMK-37: the stock firmware's key, knob and fader mapping goes back to
  its factory values.

When the unit's flash chip isn't listed: the firmware doesn't support this
unit's flash. It would run without saving anything and couldn't be updated
again except from the chip's download mode. Nothing is written.

Before writing the boot area: a power cut while it is written leaves only the
chip's ROM download mode, which on an FM-1 can need extra hardware.

## 5. Making images

- Make images from builds, not from a unit's backup. A backup holds the unit's
  pairings and settings, and on an SMK-37 its calibration.
- The sectors are the build's `flash.bin` as it is, encrypted for chip key
  `0x980F`, plus whatever else the firmware keeps outside the app (resources,
  default data).
- List every flash chip the firmware's own flash code works with, on any
  board, not the part one board happens to carry: the flash is a die in the
  chip's package, and which maker's die a batch gets can change on either
  board. Seen so far: `0x856014` (Puya P25Q80H, FM-1s), `0xB36014` (UCUN
  UC25HQ80, FM-1s from late 2026) and `0xEB6014` (SMK-37s), all 1 MiB with the
  same commands. M-VAVE's stock firmware uses JieLi's driver, which has no list
  of parts: list all three.
- Don't include the JieLi SDK's radio records (VM ids 106, 107, 108, 187).
  Firmware writes them on its first start.
- An image of M-VAVE's stock firmware needs the stock sounds (USR: FM-1
  `0xEA000`–`0xFBFFF`, SMK-37 `0xF4000`–`0xFDFFF`), which M-VAVE's packages
  don't carry.
- A release SHOULD NOT put data where a predecessor it keeps data from holds
  user data. Vacate in one release, reuse in a later one.

A reference tool converts between an image and a 1 MiB flash image (blank
outside the scope), so that images can be inspected and compared with backups
using ordinary tools.

## 6. Not in this version

- Compression, and storing the app area decrypted (docs/findings.md says why).
- Moving data between layouts, and code run by the writer before or after an
  install. The firmware moves data on its first start.
- Partial data installs (sound packs, presets): they belong to each firmware's
  own tools.
- Signatures.

## 7. Open questions

- A unit with a chip key other than `0x980F`, if one ever turns up: its images
  would need their own builds, as `.fwsc` packages would.
- SMK-37 wheel values in the stock presets header (`0xC2060`, `0xC2080`): most
  units that ran other firmware have lost them already.
- Whether the web route should get a MIDI mode command to write the boot area,
  or keep using m-bootloader's own installer package.
- The file extension: `.cfw`.
