# Clean Flash

A draft format for installing firmware on the M-VAVE FM-1 (and the Fame MiniFM)
and the SMK-37 Pro / Elite: one file, a `.cfw` image, that leaves the unit in a
known state whoever installs it and whatever ran on it before.

> Draft for discussion. Nothing here is final, and nothing implements it yet.

## Why

Installing firmware today means an M-VAVE `.fwsc` package. The package writes
the app, and each firmware's own update loader may write more: resources, a
layout switch's clears, migrations. Everything else stays as the last firmware
left it. That causes two problems:

- **Dirty flashes.** Data of the previous firmware stays behind, and the next
  one reads it as its own: wrong presets, no sound, every key one note, a unit
  that doesn't start.
- **Installs depend on the route.** A package installed through its own loader
  (M-UPGRADE, a web installer) and the same package written in download mode
  give different results, because only the loader knows what else to write.
  An installer on the device itself can run no such loader.

A Clean Flash image says exactly what every sector of the firmware's part of
the flash holds, so any writer gives the same result: M-Flasher on a computer,
a web page over MIDI through m-bootloader's MIDI mode, or a program on the
device reading the image from an SD card. Nothing outside the image's scope is
touched, and nothing inside it is left as it was, apart from the user's own
data when a firmware is upgraded and the user chooses to keep it.

An image is a 4 KiB header followed by the flash sectors it writes, exactly as
they are stored in flash. Blank sectors are left out.

## Files

- [SPEC.md](SPEC.md): the format and what a writer must do.
- [docs/findings.md](docs/findings.md): what the design rests on: what's on
  real units, what the firmwares do, and measurements.

## Status

A proposal, written from the M-Flasher and m-bootloader work. Open questions
are listed at the end of the specification.
