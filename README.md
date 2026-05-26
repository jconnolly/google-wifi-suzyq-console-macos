# google-wifi-suzyq-console-macos

Serial console + OpenWrt flashing for a **Google WiFi (Gale, AC-1304)**
puck from a **Mac**, using a $7 **SuzyQ / SuzyQable** USB-C debug
adapter — without a Chromebook in the loop, and without writing a
kernel extension.

Two pieces:

1. **[Serial monitor](#install)** — `tools/gale-sniff-all` reads the
   puck's debug UART over the SuzyQ. macOS doesn't bind a TTY to it
   (vendor-class USB, not CDC-ACM), so this talks libusb directly.
2. **[Flashing OpenWrt](docs/flashing.md)** — adapted from
   [kkestell/openwrt-on-google-wifi](https://github.com/kkestell/openwrt-on-google-wifi),
   with notes on what the SuzyQ trace should look like at each step (so
   when LED-blink-watching fails you, you have an actual log).

If you're trying to debug OpenWrt on a Google WiFi puck from a Mac and
the only guides you can find assume Linux + `/dev/ttyUSB0`, this repo is
for you.

---

## What you need

1. **Google WiFi puck** (1st-gen, model **AC-1304**, board name **Gale**).
2. **SuzyQ-style USB-C debug adapter.** I bought
   [this one from `chocolateloverraj` on eBay](https://www.ebay.com/itm/316024978790)
   ($7.32, listing title *"Chromebook Debug Board / SuzyQ / SuzyQable
   adapter for Cr50 / Ti50"*). Same person maintains the
   [`gsc-debug-board`](https://github.com/ChocolateLoverRaj/gsc-debug-board)
   project, which is the open hardware behind the cable.
3. A **Mac** (tested on Apple Silicon, macOS Sequoia).
4. **Homebrew + libusb + Python 3 + pyusb.**

---

## Install

```sh
brew install libusb
python3 -m pip install --user pyusb
git clone https://github.com/<your-user>/google-wifi-suzyq-console-macos.git
cd google-wifi-suzyq-console-macos
```

That's it — no kext, no sudo, no driver signing.

---

## Wiring

```
   ┌─────────┐                                            ┌──────────┐
   │   Mac   │═══ USB-A ═══(direct, NO dock/hub)═══╗      │  Google  │
   │         │                                     ║      │   WiFi   │
   └─────────┘                                     ║      │  AC-1304 │
                                            USB-C ╚══════►│  (Gale)  │
                                          (flip both           ● blue
                                           orientations
                                           if no enum)
   SuzyQ = passive USB-A → USB-C adapter with CC-line resistors
   that put the puck's USB-C port into Debug Accessory Mode.
   The puck draws ~500 mA of 5 V from the host through the cable,
   which is enough to boot it (blue LED = good).
```

### Common gotchas

- **Plug straight into the Mac.** Going through a USB-C dock made the
  Gale silently fail to enumerate for me. Use a plain USB-A → USB-C
  dongle if your Mac has no USB-A port.
- **Flip the USB-C end.** SuzyQ only works in one orientation. If you
  get nothing on the first try, flip it and wait 3 s.
- **The puck must show a blue LED.** That confirms it's booted enough
  for the H1 security chip to expose the debug endpoints.

---

## Use

Scan the device's interfaces to make sure it enumerated:

```sh
$ tools/gale-sniff --scan
VID=0x18d1  PID=0x500f  product='Gale debug'
interfaces in active config:
  iface 0  label='EC_PD'  subclass=0x50  endpoints=2
  iface 1  label='AP'     subclass=0x50  endpoints=2
  iface 3  label='?'      subclass=0x51  endpoints=2
```

Sniff the AP (the Marvell Armada SoC running OpenWrt / U-Boot):

```sh
$ tools/gale-sniff --duration 30
[gale-sniff] iface 1 (AP). duration=30s. Ctrl-C to stop.
coreboot-60d1b1c Mon Jan  9 00:04:49 UTC 2017 bootblock start
...
```

Power-cycle the puck (unplug + replug the SuzyQ, since the cable also
delivers power) **while** the sniffer is running, to catch the boot
trace.

Try the interactive console (will fail to write on a locked CCD — see
below):

```sh
tools/gale-console        # AP
tools/gale-console --iface 0   # EC_PD
```

---

## What actually works

| Interface | Label | Read | Write | What you see |
|-----------|-------|------|-------|--------------|
| 0 | `EC_PD` | yes | **no** (timeout) | `RST EP0 NNNN` reset chatter |
| 1 | `AP`    | yes | **no** (timeout) | full `coreboot` + `depthcharge` + vboot trace on power-up |
| 3 | (none)  | yes | n/a | silent during boot — likely the flashing / update endpoint |

Writes time out because **CCD (Closed Case Debugging) is locked**. On
Cr50 / Ti50 Chromebooks you unlock CCD with `gsctool` over the same
cable; on a stock Gale puck the unlock path isn't documented. So today
this is a read-only diagnostic, which turns out to be a lot more useful
than I expected:

- confirms the puck powers on and hands off to the bootloader
- gives you the SoC, board name, and partition layout
- reveals the full kernel command line (cros_secure, dm-verity, etc.)
- catches early-boot vboot failures

### What we actually learned from the boot trace

[`captures/full-boot.txt`](captures/full-boot.txt) is the real prize.
Highlights from one capture:

- **SoC: Qualcomm IPQ4019** (depthcharge picks `conf@7` with compat
  `google,gale-v2 qcom,ipq4019`).
- **Board: `google,gale-v2`** — this is the 1st-gen Google WiFi puck.
- **Firmware: coreboot bootblock `60d1b1c`, built `Mon Jan  9
  00:04:49 UTC 2017`.** Original factory firmware, untouched.
- **vboot kernel verification:** the signed kernel's key block signature
  fails (`In RSAVerify(): Padding check failed!`) but vboot then falls
  back to "hash only" and `Key block valid: 0` — recovery / dev-signed
  kernel accepted. The puck is in dev mode.
- **eMMC chip:** Manufacturer `000015`, product `4FTE4R`, revision
  `0.1`, serial `2789407485`.
- Boot lands at the **ChromeOS "Developer Console"** banner, which
  helpfully tells us the next step is `enable_dev_usb_boot`.

That's enough to start flashing OpenWrt — see
[`docs/flashing.md`](docs/flashing.md).

PRs welcome if you figure out the CCD unlock for Gale (would give us
interactive AP shell + EC commands). See
[`docs/ccd-unlock-research.md`](docs/ccd-unlock-research.md) for what
we know so far.

---

## Flashing OpenWrt

See [`docs/flashing.md`](docs/flashing.md). That doc combines the
well-tested USB-boot procedure from
[kkestell/openwrt-on-google-wifi](https://github.com/kkestell/openwrt-on-google-wifi)
+ [papdee's OpenWrt forum post](https://forum.openwrt.org/t/finally-installed-openwrt-on-my-google-wifi-ac-1304/183541/2)
with notes on what the SuzyQ trace should show at each step.

---

## Why this exists

Every "OpenWrt on Google WiFi" guide assumes you'll either:

- use SSH over the network (fine, but useless when networking is
  broken — which is exactly when you need a serial console), or
- use `/dev/ttyUSB0` from Linux (fine if you have a Linux box).

If you're on a Mac and your puck is in a weird state, you'd otherwise
have to dig out a Linux laptop or open the case and solder onto the
internal UART pads. This tool skips both.

---

## Related

- [chromium / hdctools — Closed Case Debugging (CCD) docs](https://chromium.googlesource.com/chromiumos/third_party/hdctools/+/HEAD/docs/ccd.md)
- [MrChromebox unbrick-with-SuzyQ guide](https://docs.mrchromebox.tech/docs/support/unbricking/unbrick-suzyq.html)
- [ChocolateLoverRaj — gsc-debug-board (the open hardware)](https://github.com/ChocolateLoverRaj/gsc-debug-board)
- [McGarrah — Google WiFi running OpenWrt](https://mcgarrah.org/google-wifi-with-openwrt/)
- [OpenWrt forum — AC-1304 install thread](https://forum.openwrt.org/t/finally-installed-openwrt-on-my-google-wifi-ac-1304/183541)
- [kestell.org — OpenWrt on Google WiFi notes](https://kestell.org/notes/networking/openwrt-on-google-wifi)
- [erichVK5 — SuzyQ PCB schematic](https://github.com/erichVK5/erichVK5-suzy-Q-cable-v1)

## License

MIT — see [LICENSE](LICENSE).
