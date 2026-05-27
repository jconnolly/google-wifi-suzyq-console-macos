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

## 2026-05-26 update — interface 3 IS the SPI bridge (USB_SUBCLASS_GOOGLE_SPI)

Decoded the subclass values from `chip/stm32/usb_descriptor.h` in
[coreboot/chrome-ec](https://github.com/coreboot/chrome-ec):

| Subclass | Name                       | What it is                         |
|----------|----------------------------|------------------------------------|
| `0x50`   | `USB_SUBCLASS_GOOGLE_SERIAL` | UART pass-through (our iface 0, 1) |
| `0x51`   | `USB_SUBCLASS_GOOGLE_SPI`    | **SPI flash chip access** (our iface 3) |
| `0x52`   | `USB_SUBCLASS_GOOGLE_I2C`    | I²C (not exposed on Gale)          |
| `0x53`   | `USB_SUBCLASS_GOOGLE_UPDATE` / `_CR50` | Firmware update / `gsctool` channel (not exposed on Gale) |

So interface 3 on Gale is the **raiden SPI bridge** — the same protocol
`flashrom`'s `raiden_debug_spi` driver speaks. If accessible, it lets
us READ AND WRITE the SPI flash chip directly through the SuzyQ,
software-only, no CH341A hardware.

Wrote a probe ([`tools/gale-spi-probe`](../tools/gale-spi-probe)) and
ran it against a walled puck. Result:

```
interface 3: subclass=0x51 (GOOGLE_SPI ✓)
[v1 xfer] JEDEC_ID: OUT 01039f (write_count=1, read_count=3)
  IN 05009f9f9f  status=0x0005  payload=3B
  ERROR status 0x0005
```

`bInterfaceProtocol = 0x01` = USB-SPI **v1** (older 2-byte header
format, not the v2 packet-ID format).

`status=0x0005` is the defined error code from `chip/stm32/usb_spi.h`:

> 0x0005: The SPI bridge is disabled.

This is a HUGE positive signal:
- Writes to iface 3 are NOT blanket-blocked (unlike UART writes which
  time out).
- The device speaks the raiden v1 protocol.
- It returns a *defined* error code, not silence or garbage.
- The only thing missing is "the SPI bridge has been enabled."

The wedge is clear: find the Gale-specific way to flip the "SPI bridge
enabled" bit.

### Why the standard Cr50 unlock won't work

On a Chromebook with Cr50, you unlock CCD (and enable the SPI bridge
as one of its capabilities) by running `gsctool` from ChromeOS shell.
`gsctool` talks to the GSC over `USB_SUBCLASS_GOOGLE_UPDATE` (`0x53`).

**Gale does not expose a `0x53` interface.** Only `0x50` (UART) and
`0x51` (SPI). So the gsctool unlock path is structurally unavailable.

### Open questions to research

1. Does the H1 on Gale accept a CCD-unlock-equivalent command in-band
   over UART (iface 0 EC_PD console)? UART writes currently time out
   too, but maybe a specific magic string triggers an unlock-prompt
   sequence (similar to Cr50's `ccd open` over its own console).
2. Is there a physical write-protect screw on the Gale mainboard that
   needs to be removed to allow CCD to be opened? On Chromebooks this
   is a SOIC-8-area screw. The 2026-05-22 memory note speculates this
   exists.
3. Could SET_INTERFACE alt-setting toggle on iface 3 enable the
   bridge? Unlikely (no alt settings shown in descriptors) but cheap
   to try.
4. Is there a vendor-specific control transfer (bRequest range
   0xA0-0xFF) that flips the CCD state? Most GSCs use control
   transfers for capability management.
5. Could we extract the H1 firmware via JTAG / SWD pads on the
   mainboard and patch the CCD-lock check out? Same answer as
   "CH341A on the AP SPI" — needs hardware + extracted firmware
   keys + RE work.

### Why this matters

If we can enable the SPI bridge on a walled puck, we can:
1. Dump the puck's coreboot/AP firmware via SuzyQ alone (no CH341A)
2. Compare it to a non-walled puck's firmware (e.g. mesh1) to
   identify the kernel-signature-enforcement diff
3. Reflash the walled puck with a non-walled image (or patch out the
   sig check), restoring OpenWrt USB-boot ability

That's the entire CH341A workflow done in software, through the $7
cable. Huge win for the project.

## Status

Open. PRs welcome. The SPI-bridge-disabled finding is the wedge —
anyone who knows the Gale H1 CCD unlock path would unblock the rest.
