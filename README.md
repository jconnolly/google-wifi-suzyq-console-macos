# google-wifi-suzyq-console-macos

Serial console for a **Google WiFi (Gale, AC-1304)** puck from a **Mac**,
using a **SuzyQ / SuzyQable** USB-C debug adapter — without a Chromebook
in the loop, and without writing a kernel extension.

If you're trying to debug OpenWrt on a Google WiFi puck and the only guides
you can find assume Linux + `/dev/ttyUSB0`, this repo is for you. macOS
doesn't bind a TTY to the Gale's debug interfaces because they're
vendor-class, not CDC-ACM. So we talk libusb directly.

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
| 0 | `EC_PD` | yes | **no** (timeout) | `RST EP0 NNNN`, `TPM: command …` |
| 1 | `AP`    | yes | **no** (timeout) | `coreboot-… bootblock start` on power-up, silent after |
| 3 | (none)  | yes | n/a | small binary blob — likely flashing / update endpoint |

Writes time out because **CCD (Closed Case Debugging) is locked**. On
Cr50 / Ti50 Chromebooks you unlock CCD with `gsctool` over the same
cable; on a stock Gale puck the unlock path isn't documented. So today
this is a read-only diagnostic, which is still useful:

- confirm the puck powers on and hands off to the bootloader
- confirm which firmware revision is on the H1
- catch early-boot panics before the Marvell SoC silences the console

PRs welcome if you figure out the unlock.

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
