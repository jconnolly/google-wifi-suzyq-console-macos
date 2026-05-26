# Blog post notes — getting a serial console on a Google WiFi puck from a Mac

Working title candidates:

- "I bought a $7 cable on eBay to debug a Google WiFi puck from my Mac"
- "Serial console on a Google WiFi (AC-1304) from macOS, no Chromebook needed"
- "Debugging an OpenWrt-flashed Google WiFi puck over SuzyQ — the macOS path"

## Hook / lede

I'm flashing six Google WiFi pucks (model AC-1304, codename **Gale**) over to
OpenWrt. The USB-boot pilot worked; the next puck is stuck somewhere in
"can't get Mac ↔ puck ethernet link". With networking unreliable I needed a
serial console — and every guide assumed Linux + `/dev/ttyUSB0`. I'm on a
Mac.

## What I bought

A "SuzyQ / SuzyQable" USB-C debug adapter from
[chocolateloverraj on eBay](https://www.ebay.com/itm/316024978790) for $7.32.
The same person publishes the open hardware design at
[gsc-debug-board](https://github.com/ChocolateLoverRaj/gsc-debug-board).
2,197 sold at the time I bought one, which is reassuring.

A SuzyQ is a passive USB-A → USB-C adapter with specific resistors on the
CC lines that put the target's USB-C port into **Debug Accessory Mode**.
The target then exposes a USB device with bulk endpoints for the various
on-board consoles (AP, EC, etc.) — this is the "Closed Case Debugging"
pattern Google uses on Chromebooks and a handful of other devices (Gale
included).

## Things that surprised me

1. **The Gale has only one USB-C port.** Power and debug come through the
   same connector. SuzyQ supplies ~500 mA at 5 V over the cable, which
   turned out to be enough to boot the puck. The blue LED is the tell.
2. **Going through a USB-C dock killed it.** The puck enumerated nothing
   for ages. Plugging the SuzyQ directly into the Mac via a plain USB-A
   → USB-C dongle made it work instantly.
3. **macOS does not give you a `/dev/cu.usbmodem*`.** The Gale debug
   interfaces are vendor-class (`bInterfaceClass=0xFF`), not CDC-ACM, so
   macOS doesn't auto-bind a TTY. You have to talk libusb directly. ~40
   lines of pyusb does it.
4. **CCD is locked on a stock Gale.** I can read every interface, I
   cannot write to any of them — every OUT write times out. That's
   enough to confirm power, capture a boot trace, and watch the firmware
   panic, but you can't type at a shell.

## The actual capture

```
$ python3 tools/gale-sniff --scan
VID=0x18d1  PID=0x500f  product='Gale debug'
interfaces in active config:
  iface 0  label='EC_PD'  subclass=0x50  endpoints=2
  iface 1  label='AP'     subclass=0x50  endpoints=2
  iface 3  label='?'      subclass=0x51  endpoints=2

$ python3 tools/gale-sniff --duration 90
[gale-sniff] iface 1 (AP). duration=90s. Ctrl-C to stop.
coreboot-60d1b1c Mon Jan  9 00:04:49 UTC 2017 bootblock start
[gale-sniff] captured 65 bytes
```

One line. That's the entire AP output before the H1 hands off and the
console goes dark. Useful: tells me the puck is alive, the H1 is doing
its job, and the bootblock from Jan 2017 is intact.

## What's next (post-blog)

- Figure out the CCD unlock for Gale (the Cr50 `gsctool` flow doesn't
  translate directly; need to RE the H1's CCD state machine).
- See if interface 3 (subclass 0x51) is a flashing endpoint.
- Document the actual Mac-↔-puck ethernet link issue once I've got
  serial to look at it.

## Photo ideas

- Mac + puck + SuzyQ on the bench
- screenshot of `gale-sniff --scan`
- the coreboot line in a terminal
- the eBay listing for context (or photo of the actual cable next to a
  ruler)

## Outline

1. Problem: I'm flashing 6 Google WiFi pucks to OpenWrt, networking
   misbehaves, need serial.
2. Why every guide is useless on a Mac (Linux assumptions, vendor-class
   USB).
3. The $7 cable from eBay.
4. The setup that actually worked (and the 3 things that didn't).
5. Read the boot trace, confirm the puck is alive.
6. Why I can read but not write (CCD lock).
7. Code: ~40 lines of pyusb, dropped on GitHub.
8. Where to go next.
