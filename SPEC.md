# Clean Flash image format, draft 1

A Clean Flash image (`.cfw`) describes the contents of part of a unit's flash:
the firmware, the bootloader, or both. A writer installs it so that every
sector in that part ends up exactly as the image says, apart from the user's
data on an upgrade the user chose to keep.

The words MUST, MUST NOT, SHOULD and MAY are used as in RFC 2119.

## 1. Units and terms

| Board | Units | Flash |
|---|---|---|
| 1, FM-1 | M-VAVE FM-1, Fame MiniFM | 1 MiB, 256 sectors of 4 KiB |
| 2, SMK-37 | M-VAVE SMK-37 Pro, SMK-37 Elite | 1 MiB, 256 sectors of 4 KiB |

- **Sector:** 4 KiB of flash, at an address that is a multiple of `0x1000`.
  Sector *n* starts at `n × 0x1000`.
- **Boot area:** sectors 0 to 3 (`0x00000` to `0x03FFF`): the flash header, the
  boot file table, the bootloader and its configuration, including the chip-key
  record.
- **Chip key:** the 16-bit key the chip decrypts the app area with. Every unit
  seen so far uses `0x980F`.
- **App area encryption (SFC):** the app area, from `0x4000`, is stored
  encrypted. Each 32-byte block at flash address *a* is ENC-ciphered with the
  key `chip_key ^ ((a − 0x4000) >> 2)`; the last block may be partial. ENC is
  JieLi's byte cipher: for each byte, `out = in ^ (key & 0xFF)`, then
  `key = (key << 1) ^ (0x1021 if key & 0x8000 else 0)`, 16 bits.
- **Writer:** whatever installs an image: M-Flasher in download mode, a web
  page through m-bootloader's MIDI mode or RAM stage, a program on the device.
- **Packager:** whoever makes an image: a firmware's author, or anyone
  repackaging an open-source firmware.

## 2. Scopes and protected ranges

An image has one **scope**, which fixes the sectors it writes. A writer writes
every sector in the scope and no other. There is no way to leave a sector in
the scope as it was, except the user data kept on an upgrade (section 4).

| Scope | FM-1 | SMK-37 |
|---|---|---|
| 1, firmware | `0x04000`–`0xFEFFF` | `0x04000`–`0xBFFFF`, `0xC2000`–`0xFEFFF` |
| 2, bootloader | `0x00000`–`0x03FFF` | `0x00000`–`0x03FFF` |
| 3, bootloader and firmware | both of the above | both of the above |

**Protected ranges** are in no scope. No writer writes them, whatever the image
holds:

| Board | Range | Holds |
|---|---|---|
| both | `0xFF000`–`0xFFFFF` | key area (`key_mac`): factory Bluetooth address and flash efuse copy where present; m-bootloader's calibration copy (`TRIMCOPY`) on the SMK-37 |
| SMK-37 | `0xC0000`–`0xC0FFF` | BTIF: the Bluetooth address and pairings, in the JieLi SDK's format |
| SMK-37 | `0xC1000`–`0xC1FFF` | USRTRIM: the factory key, pad and wheel calibration (`trimdat`) |

A range is protected only if every known firmware either never writes it or
uses it in one shared format. New scopes and protected ranges need a new format
version.

## 3. File layout

All integers are little endian. Offsets are from the start of the file. A file
is at most 4 MiB.

| Part | Where |
|---|---|
| Header | offset 0, 512 bytes |
| Section table | where the header says |
| Sections | where the section table says |

### 3.1 Header

| Offset | Size | Field |
|---:|---:|---|
| 0 | 4 | Magic: `43 46 57 1A` (`CFW`, then `0x1A`) |
| 4 | 2 | Format version: 1 |
| 6 | 2 | Header size: 512 |
| 8 | 1 | Board (section 1) |
| 9 | 1 | Scope (section 2) |
| 10 | 2 | Flags, all reserved: 0 |
| 12 | 4 | Flash size: `0x100000` |
| 16 | 4 | Encrypted end: the app area is SFC-encrypted from `0x4000` up to this address (exclusive). 0: nothing is encrypted |
| 20 | 4 | File size |
| 24 | 4 | Offset of the section table |
| 28 | 2 | Number of sections |
| 30 | 2 | Reserved: 0 |
| 32 | 32 | SHA-256 of bytes 512 to the end of the file |
| 64 | 32 | Name, UTF-8, padded with zeros: shown on the device's screen |
| 96 | 16 | Version, UTF-8, padded with zeros |
| 112 | 396 | Reserved: 0 |
| 508 | 4 | CRC-32 of bytes 0 to 507 |

CRC-32 is the IEEE polynomial as in zlib (reflected, initial and final value
`0xFFFFFFFF`), the check the chip computes on a flash range.

### 3.2 Section table

One 16-byte entry per section:

| Offset | Size | Field |
|---:|---:|---|
| 0 | 2 | Type |
| 2 | 2 | Flags. Bit 0: must be understood |
| 4 | 4 | Offset |
| 8 | 4 | Length |
| 12 | 4 | CRC-32 of the section |

A reader skips a section of a type it doesn't know, unless the section is
marked must be understood: then it refuses the file. Each type appears at most
once.

| Type | Section | Required |
|---:|---|---|
| 1 | Sector table | yes |
| 2 | Sector data | yes |
| 3 | User map | no |
| 4 | Predecessors | no |
| 5 | Requirements | no |
| 6 | Text | no |

### 3.3 Sector table (type 1)

256 entries of 12 bytes, one per sector of the flash, in address order:

| Offset | Size | Field |
|---:|---:|---|
| 0 | 4 | Offset of the sector's data, from the start of the sector data section |
| 4 | 2 | Stored length in bytes |
| 6 | 1 | Kind |
| 7 | 1 | Flags. Bit 0: call filter (kind 2 only) |
| 8 | 4 | CRC-32 of the sector's plain content |

| Kind | Meaning |
|---:|---|
| 0 | Erased: the sector holds 4096 bytes of `0xFF`. Nothing stored |
| 1 | Stored: 4096 bytes as they are |
| 2 | Upkr: a packed stream that unpacks to 4096 bytes |

Every sector outside the scope MUST be kind 0. A reader refuses an image whose
entries for those sectors are anything else, so that an image made from a
backup can't carry another unit's data unnoticed.

**Plain content** is the sector's 4096 bytes before encryption: what the app
reads. The CRC-32 is of the plain content, so it doesn't depend on the chip
key. A writer computes the bytes for the flash by encrypting the part of the
sector below the encrypted end (section 1) with the unit's own chip key.

**Upkr** is Upkr 0.2.3 (github.com/exoticorn/upkr) with two parity contexts,
as `upkr -9 -p 2` writes and m-bootloader's `src/upkr.c` reads. The stream
unpacks to exactly 4096 bytes; a reader refuses one that would unpack to more
or less.

**Call filter**, after unpacking: at each even offset *i* from 0 while
*i* + 4 ≤ 4096, if bytes *i*, *i*+1 are `BF EA`, the 16-bit value at *i*+2 has
*i*/2 subtracted from it (modulo 2^16) and the scan continues at *i*+4;
otherwise it continues at *i*+2. The packer applies the inverse (adds *i*/2)
before packing. It's m-bootloader's filter for pi32v2 calls, so that calls to
one function look alike.

### 3.4 Sector data (type 2)

The stored and packed sectors, one after another, in any order. No alignment
is required. The section's CRC-32 covers all of it.

### 3.5 User map (type 3)

32 bytes, bit *n* (byte *n*/8, bit *n* mod 8) for sector *n*: the sector holds
the user's data (settings, projects, presets, samples, pairings). The image
holds its default content for these sectors, often erased. Only sectors in the
firmware scope may be set; a reader refuses a map that sets any other.

### 3.6 Predecessors (type 4)

The earlier builds this image upgrades, and what is kept from each. The
packager lists them; nothing on the unit has to cooperate.

| Offset | Size | Field |
|---:|---:|---|
| 0 | 2 | Number of entries |
| 2 | 2 | Reserved: 0 |
| 4 | | Entries, 96 bytes each |

Entry:

| Offset | Size | Field |
|---:|---:|---|
| 0 | 4 | Fingerprint start: a flash address |
| 4 | 4 | Fingerprint length, more than 0 |
| 8 | 4 | CRC-32 of that range as stored in flash |
| 12 | 2 | The chip key the CRC was computed for |
| 14 | 2 | Reserved: 0 |
| 16 | 32 | Keep map: sectors kept on an upgrade from this build, as in the user map |
| 48 | 48 | Label, UTF-8, padded with zeros, for example `Melodee 1.6.0` |

The fingerprint range SHOULD cover only that build's code, not its data, so
that the same build matches whatever the user stored. The keep map MUST be a
subset of the image's user map.

### 3.7 Requirements (type 5)

A list of items: a 2-byte type, a 2-byte length, then that many bytes.

| Type | Contents | A writer whose unit doesn't meet it |
|---:|---|---|
| 1 | Flash chip JEDEC IDs the image runs on, 4 bytes each | refuses |
| 2 | CRC-32s of boot areas (sectors 0–3 as stored) the firmware is known to start with, 4 bytes each | warns |

### 3.8 Text (type 6)

UTF-8 lines of `key=value`, for hosts to show: `name`, `version`, `author`,
`url`, `license`, `notes`. A device MAY ignore it.

## 4. Installing

### 4.1 Checks before anything is written

A writer MUST check all of these, and MUST refuse the image, with nothing
written, if one fails:

1. The magic, the format version, the header's CRC-32, and the board matching
   the unit.
2. On a host: the SHA-256. On every writer: each section's CRC-32.
3. Every sector entry: within the sector data; kind 0 outside the scope;
   packed streams unpack to 4096 bytes; the plain content's CRC-32 matches.
   A writer that can't hold the whole image does this in a first pass over
   the file and keeps the 256 CRCs.
4. The user map and keep maps within the firmware scope.
5. Requirement type 1, if present.
6. Scopes 2 and 3: the boot area is a bootloader the writer recognises for
   this board, and its chip-key record decodes to the unit's chip key, read
   from the chip.

### 4.2 Clean install or keep

For scopes 1 and 3, the writer looks for a predecessor: the first entry whose
fingerprint range, read from the unit's flash, has the listed CRC-32 and whose
chip key is the unit's. Writers can use the chip's own CRC-32 of a range
(m-bootloader's MIDI command `0x24`, M-Flasher's loader).

- **A predecessor matches:** the writer asks whether to keep the user's data,
  naming the predecessor's label. Keeping is the default. If the user keeps
  it, the sectors in that entry's keep map are left as they are. Every other
  sector in the scope is written.
- **None matches:** a clean install. Every sector in the scope is written. The
  writer says what that means (section 4.5) before anything is written.

The new firmware converts kept data on its first start, as firmwares do across
their own updates; that is the firmware's job, not the writer's.

### 4.3 Writing

- A host SHOULD save a backup of the whole flash before the first write.
- A sector whose flash content already matches is skipped. A writer compares
  the chip's CRC-32 of the sector with the CRC-32 of the bytes it would write.
- Every written sector is verified: read back, or the chip's CRC-32 of it.
- **Scope 2 and 3, the boot area first:** sectors 3, 2 and 1, then sector 0
  last, so that a boot area cut short has no valid file table.
- **Scope 1 and 3, the firmware:** erase `0x4000` first, then write every
  other sector in the scope, then `0x4000` last, so that until the end nothing
  in flash can pass as firmware.
- A writer MUST NOT write a protected range, and SHOULD check that a second
  time where it erases.

### 4.4 Routes

| Writer | Firmware scope | Boot area |
|---|---|---|
| M-Flasher, download mode | writes the plan itself | yes, after a backup and its own confirmation |
| Web page, m-bootloader MIDI mode or RAM stage | builds an ordinary package from the planned flash and serves it: MIDI mode writes `0x4000` up to the package's end, skips sectors that already match, and never writes the boot area. For kept sectors the page first reads them from the unit (`0x23`) and serves them back | no |
| The device: a RAM app reading the image from an SD card (m-bootloader's SD card item) | yes, in two passes: check, then write | as M-Flasher, if the app implements it |

### 4.5 What the writer tells the user

Before a clean install:

- The data of the firmware installed now (settings, projects, presets,
  samples) is erased. A backup brings it back.
- Bluetooth pairings are erased. On an FM-1 the unit also gets a new Bluetooth
  address, so computers and phones see a new device.
- On an SMK-37: the stock firmware's key, knob and fader mapping goes back to
  its factory values.

Before writing the boot area: a power cut while it is written leaves only the
chip's ROM download mode, which on an FM-1 can need extra hardware.

## 5. Making images

- Make images from builds, not from a unit's backup. A backup holds the unit's
  pairings and settings, and on an SMK-37 its calibration.
- Store the app area decrypted (the plain content) and set the encrypted end.
  Encrypted code doesn't compress, and a decrypted image fits any chip key.
- Don't include the JieLi SDK's radio records (VM ids 106, 107, 108, 187).
  Firmware writes them on its first start.
- An image of M-VAVE's stock firmware needs the stock sounds (USR: FM-1
  `0xEA000`–`0xFBFFF`, SMK-37 `0xF4000`–`0xFDFFF`), which M-VAVE's packages
  don't carry.
- A release SHOULD NOT put data where a predecessor it keeps data from holds
  user data. Vacate in one release, reuse in a later one.

## 6. Not in this version

- Moving data between layouts, and code run by the writer before or after an
  install. The firmware moves data on its first start.
- Partial data installs (sound packs, presets): they belong to each firmware's
  own tools.
- Signatures. A section type can be added for them later.

## 7. Open questions

- FM-1s with the new flash chip (`0xB36014`): whether the boot configuration
  of today's bootloaders suits them.
- SMK-37 wheel values in the stock presets header (`0xC2060`, `0xC2080`): kept
  only if a writer offers it; most units that ran other firmware have lost
  them already.
- Whether the web route should get a MIDI mode command to write the boot area,
  or keep using m-bootloader's own installer package.
- Whether a predecessor should be able to name another firmware family whose
  data it reads.
- The file extension: `.cfw`.
