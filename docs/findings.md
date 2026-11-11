# What the design rests on

Notes from the M-Flasher and m-bootloader work, November 2026: real units'
flash images, the official packages, and the sources of the custom firmwares.
"Images" below are full 1 MiB flash reads: 68 distinct ones, from one SMK-37
Pro and one or more FM-1s, taken over several weeks while different firmwares
were installed.

## The boot area holds nothing unit-specific

Every official package carries a fixed boot area:

| Packages | Boot area (`0x0000`–`0x3FFF`) |
|---|---|
| FM-1 009, 014, 015; SMK-37 Pro 016; SMK-37 Elite 016 | identical, byte for byte: bootloader `0x3830` bytes, `isd_config` `0x2BB` bytes |
| FM-1 016 | another fixed one: bootloader `0x3898` bytes, `isd_config` `0x2F3` bytes |

Every image holds either one of those two boot areas, byte for byte, or an
m-bootloader install (versions 1.1 to 2.1), on both boards, including units
running Baud Girl and Snoop37.
The parts m-bootloader 2.x keeps from the unit when it installs itself (the
flash header, the boot table's entries from `0x60`, the 34-byte chip-key
record) are the same in all 68 images and all packages. So a bootloader can
ship a fixed boot area, as M-VAVE's packages do. The one check worth keeping is
that the chip-key record matches the chip.

## The FM-1's flash holds nothing unit-specific

The FM-1's key area (`0xFF000`) is blank in the packages and on the units seen.
With no factory address there, the firmware makes up a Bluetooth address on
its first start and keeps it in BTIF (stock 015: `0xE9000`; 016 moves it).
After a clean install the unit gets a new address.

## Radio calibration is rebuilt by firmware

From fm1-base's `docs/bt/RF-CALIBRATION.md` and `BLE-HW-FACTS.md`, checked
against the images: the JieLi SDK keeps the radio's settings as VM records.
106 (crystal trims), 107 (PA trims) and 108 (Wi-Fi gains) are board constants
written at first start: every image holds the SDK's demo values
(`0B 0B`, `01 07 04 07 0B 01 07`). 187 is the radio's own measurement, made
again when missing. Nothing is read from efuse. fm1-base's own Bluetooth and
Wi-Fi code calibrate at every start. Erasing these records costs a slightly
longer first start.

## The SMK-37

- **USRTRIM** (`0xC1000`, `trimdat` at `+0x224`): the factory key, pad and
  wheel curves. Every custom firmware read treats it as read-only or ignores
  it: genkigenki's family (Speak37, Makkuro37, Vector37, erik37), Skiff,
  ChoralRoot, ChanceTheMaker's. M-VAVE's loader has been seen to erase it when
  staged the wrong way, and m-bootloader's calibration guard keeps a copy at
  `0xFF000` (`TRIMCOPY`) and puts it back.
- **BTIF** (`0xC0000`): no custom firmware writes it. Stock firmware keeps the
  Bluetooth address there in the JieLi SDK's format, and M-VAVE's open SMK-37
  firmware leaves it to the SDK (its flash API excludes it). On the unit seen
  it held five different addresses over time and was blank in seven images:
  each time it was erased, the SDK made a new address.
- **Wheels:** M-VAVE's open SMK-37 firmware finds the wheels' rest positions at
  every start, and genkigenki's layer and Skiff calibrate them themselves. Only
  stock firmware uses the wheel values in its presets header (`0xC2060`,
  `0xC2080`).
- **USRFLASH** (`0xC2000`–`0xEAFFF`): the stock firmware's key, knob and fader
  mapping, trusted while its end marker (`usrdata` at `0xEAFF8`) is there.
  ChoralRoot, Makkuro37, Vector37 and Skiff store their own data there; the
  stock firmware rebuilds a factory mapping when the marker is gone. ChoralRoot
  once left that marker over its own data, and stock firmware then sent one
  note for every key: a dirty flash.

## How firmwares move data today

Every precedent moves data in the new firmware, on its first start:

- **Melodee 1.5.1 to 1.6:** preset banks move from fixed pairs at
  `0xCB000`–`0xDFFFF` into a pool (`st_small_migrate`), with a "migrating
  presets" screen. Its loader won't write the new resource region over
  unmoved banks, so updates through Firmware Hub need two installs.
- **fm1-base / LOOM-8 0.12 to 0.13:** projects move from fixed pairs into a
  pool at every start until done (`st_pool_boot`): copy, read back, erase the
  old sector. The resource region is written only after that.
- **ChoralRoot:** 0.22 to 0.29.2 imported 0.21's sounds; 3y231 retired the
  import and freed the space; 3y234 put the fonts there. Vacate, then reuse.
- **Optimist:** doesn't move; a layout switch erases what would move, after a
  "back up first" warning.

## Resources outside the app

Some firmwares keep data that their own loader or installer writes, beside the
app:

| Firmware | Where | How it's installed |
|---|---|---|
| Melodee 1.6 | `0xCB000`, from `res.bin` in the `.fwsc` | its loader, or over MIDI after the install |
| ChoralRoot 0.38, FM-1 | `0x93000`–`0x9EFFF`, inside `flash.bin` | with the app |
| LOOM-8 0.13+ | `0xAB000`, from a separate `-res.bin` | over MIDI after the install |
| Optimist 0.3 | none; layout switches clear `[app end, R)` | its loader |

In a Clean Flash image all of these are ordinary sectors.

## m-bootloader's MIDI mode already installs large images

MIDI mode (and the RAM stage) installs a package's `flash.bin` up to `0xFF000`
in two passes: it reads everything once and keeps one CRC per sector, then
reads each sector again, skips it if the flash holds it, and otherwise erases,
writes and verifies it, with `0x4000` erased first and written last. It never
holds the image in RAM, and never writes the boot area or, on the SMK-37,
`0xC0000`–`0xC1FFF`.

## Why images aren't compressed or decrypted

An earlier draft stored the app area decrypted and packed each sector with
Upkr. Measured on Melodee 1.6.1's app area (586 KB used), Upkr 0.2.3 `-9 -p 2`:

| Stored as | Size |
|---|---:|
| encrypted, as in the `.fwsc` | 567 KB (97%) |
| decrypted, one stream per 4 KiB sector, call filter | 418 KB (71%) |
| decrypted, 64 KiB streams, call filter | 398 KB (68%) |
| decrypted, one stream, call filter | 389 KB (66%) |

Encrypted code doesn't compress, so compression only pays with decryption,
and together they need a decoder in every writer (JavaScript, Rust, C), a call
filter, and encryption with the unit's key while writing. What they save is
about a third of the code, a few hundred KB of download. It doesn't help the
web route at all: m-bootloader's MIDI mode takes raw bytes, so the page would
unpack everything before sending it. So images hold sectors as stored, and
leave out blank ones instead:

| Firmware | Sectors stored (FM-1 firmware scope: 1012 KiB) |
|---|---:|
| M-VAVE FM-1 015 | 572 KiB, plus 72 KiB of stock sounds |
| M-VAVE FM-1 016 | 660 KiB |
| ChoralRoot 0.38 | 616 KiB |
| Melodee 1.6.1, with its resources | 628 KiB |
| LOOM-8 0.14, with its resources | 660 KiB |
| Optimist 0.3 synths | 648 KiB |
| Optimist 0.3 mots-config (SD card build) | 808 KiB |

A bootloader image is 20 KiB: the header and four sectors.
