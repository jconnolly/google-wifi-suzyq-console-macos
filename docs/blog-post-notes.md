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

## The actual capture (first pass — sad)

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

One line. The interesting boot output happens in the first 10 s,
*before* my sniffer was ready. Need to start the sniffer first, then
power the puck.

## The actual capture (second pass — useful)

But the puck is *powered by the SuzyQ* — unplugging the cable to
power-cycle takes the device off the USB bus too, so the sniffer can't
"already be running" in the normal sense. So I added `--wait` and
auto-reconnect: the script polls for the device to appear, then starts
reading the instant it does.

```
$ tools/gale-sniff-all --wait --duration 120
[gale-sniff-all] waiting for device... plug SuzyQ in now
# (I yank the cable, count to 3, plug back in)
[gale-sniff-all] device appeared
[gale-sniff-all] running. duration=120.0s.

18:05:26 [EC_PD] RST EP0 3220
18:05:26 [AP] coreboot-60d1b1c Mon Jan  9 00:04:49 UTC 2017 bootblock start
                     VbBootDeveloper() - trying fixed disk
18:05:26 [AP] sound_stop: No sound ops set.
18:05:26 [AP] VbTryLoadKernel() start, get_info_flags=0x2
18:05:26 [AP] MMC version  = 10000042
18:05:26 [AP] Man 000015 Snr 2789407485 Product 4FTE4R Revision 0.1
18:05:26 [AP] GptNextKernelEntry looking at new prio partition 2
18:05:26 [AP] GptNextKernelEntry likes partition 2
18:05:26 [AP] Found kernel entry at 20480 size 32768
18:05:26 [AP] Checking key block signature...
18:05:26 [AP] In RSAVerify(): Padding check failed!
18:05:26 [AP] Verifying key block signature failed.
18:05:26 [AP] Checking key block hash only...
18:05:26 [AP] Kernel preamble is good.
18:05:28 [AP] In recovery mode or dev-signed kernel
18:05:28 [AP] TPM: Lock physical presence
18:05:28 [AP] Modified kernel command line: cros_secure console= loglevel=7
                init=/sbin/init ... root=PARTUUID=cc24514c-... dm_verity...
18:05:28 [AP] Loading FIT.
18:05:28 [AP] Config conf@7, kernel kernel@1, fdt fdt@7,
                compat google,gale-v2 (match) qcom,ipq4019
18:05:28 [AP] Choosing best match conf@7.
18:05:28 [AP] Exiting depthcharge with code 4 at timestamp: 42069715
18:05:35 [AP] Developer Console
18:05:35 [AP] To use this console, the developer mode switch must be engaged.
18:05:35 [AP] - install your own operating system image!
18:05:35 [AP] If you are having trouble booting a self-signed kernel, you may
                need to enable USB booting.  To do so, run the following as root:
18:05:35 [AP]   enable_dev_usb_boot
18:05:35 [AP] Have fun and send patches!
```

That's the whole boot. From this single trace I learned:

- **SoC: Qualcomm IPQ4019.** I'd been telling myself this was Marvell.
  Wrong. Depthcharge tells me unambiguously that it picks `conf@7` with
  `compat = "google,gale-v2 qcom,ipq4019"`.
- **eMMC: Manufacturer 000015, product 4FTE4R, serial 2789407485.** A
  Samsung 4 GB eMMC. The serial number doubles as a unique device ID for
  inventory.
- **vboot does its thing:** RSA padding check fails (because the kernel
  is dev-signed, not factory-signed), then falls back to hash-only and
  accepts it. So the puck is already in dev mode — I don't need to
  trigger anything to allow custom kernels.
- **Kernel command line in the clear.** `cros_secure console= ... dm_verity ...
  kern_guid=cc24514c-...` — that PARTUUID is what depthcharge handed off
  to the kernel. Useful if I want to mess with the GPT layout.
- **Boot lands at the ChromeOS Developer Console banner**, which
  literally tells me to run `enable_dev_usb_boot` to USB-boot a custom
  image. So the path forward is exactly what kkestell's guide already
  describes — but now I have first-hand confirmation that THIS puck is
  in the expected state.

Without serial, all of the above would have been guesswork from LED
colors and ping responses.

## What's next (post-blog)

- Actually flash this puck — the serial trace was the diagnostic
  needed to be confident before the irreversible `dd` to `/dev/mmcblk0`.
  Following [`docs/flashing.md`](flashing.md).
- Figure out the CCD unlock for Gale (the Cr50 `gsctool` flow doesn't
  translate directly; need to RE the H1's CCD state machine). See
  [`docs/ccd-unlock-research.md`](ccd-unlock-research.md).
- See if interface 3 (subclass 0x51) is a flashing endpoint.
- Document the actual Mac-↔-puck ethernet link issue once I've got
  serial to look at it.

## The "didn't take no for an answer" arc

The shape of the story matters more than the raw technical content:

1. Mac sees nothing on the USB bus when I plug the SuzyQ in. → fixed by
   bypassing my USB-C dock and going direct.
2. Device enumerates as "Gale debug" but no `/dev/cu.usbmodem*`. → not a
   driver problem to fix; it's vendor-class USB, so write 40 lines of
   pyusb instead.
3. AP UART is silent on idle puck. → not a bug; serial only TXes during
   boot, so power-cycle while sniffing.
4. But power-cycling drops the device. → add `--wait` + reconnect loop.
5. Sniffer captures only 1 line. → sniffer started too late; flip the
   order: start sniffer first, replug for power-on second.
6. **Full boot trace.** Tells me everything I need.

Every step had a "give up and use a Linux laptop / open the case /
solder onto pads" branch. Following each unblock instead is what makes
this a blog post.

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
