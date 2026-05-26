# Notes on CCD unlock for Gale (write access via SuzyQ)

Today the SuzyQ adapter gives us **read-only** access to all 3 of the
Gale's debug interfaces. Every write times out. We'd like to figure out
how to unlock the write path so we can:

- type into the `depthcharge` shell during boot (bypass needing SW7)
- inspect / modify CCD capabilities like a Chromebook can
- in theory: enable the `enable_dev_usb_boot` flag without opening the
  case

This file collects what we know and what's still unknown.

## What we know

- The Gale's debug device enumerates as `Google Inc. (0x18d1) / "Gale
  debug" (0x500f)`.
- Three vendor-class (`bInterfaceClass=0xFF`) interfaces:
  - `iface 0` — labelled `EC_PD`, subclass `0x50`
  - `iface 1` — labelled `AP`, subclass `0x50`
  - `iface 3` — unlabelled, subclass `0x51`
- Each interface has one IN + one OUT bulk endpoint.
- Reads work. Writes (across all three) return `LIBUSB_ERROR_TIMEOUT`.
- The security chip on the puck is the **Google H1** (Citadel family —
  precursor to Cr50). It mediates CCD.
- The familiar Chromebook CCD unlock involves `gsctool` over the same
  cable, but the procedure is documented for **Cr50** / **Ti50**, not
  for H1 on Gale.

## What's worth trying

- Run `gsctool -c` against `/dev/cu.usbmodem*` — won't work directly
  because macOS doesn't bind a TTY, but the source could be adapted to
  talk libusb to interface 3 (subclass `0x51` is suspicious — it might
  be the CCD/flash command endpoint).
- Inspect [chromium hdctools](https://chromium.googlesource.com/chromiumos/third_party/hdctools/)
  for any Gale / H1 specific code paths.
- The
  [ChocolateLoverRaj/gsc-debug-board](https://github.com/ChocolateLoverRaj/gsc-debug-board)
  hardware repo (same person who sells the eBay adapter we used) may
  have debug notes specifically about H1 vs Cr50 vs Ti50 unlock
  differences.
- "Physical presence" assertion on Chromebooks is the power button. On
  the Gale puck, the equivalent might be the external reset button
  and/or SW7 — possibly with a USB request sequence layered on top.

## Where this would help

If we get write access:

- step 2 of `docs/flashing.md` could become "type a command at the
  depthcharge shell" instead of "press SW7 twice with the right
  timing".
- recovery of a partially-bricked puck without opening the case.
- diagnostics on a puck whose ethernet isn't coming up (you can't SSH
  in to read logs, but you'd be able to talk to the kernel via serial).

## Status

Open. PRs welcome.
