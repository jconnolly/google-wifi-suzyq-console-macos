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

## Two pucks, two firmware generations — the "wall" theory in serial form

Captured boot traces on two pucks back-to-back. They are visibly
different firmware generations, which lines up with the community
hypothesis that auto-updated pucks (running online 24/7) get a newer
Google firmware that won't sustain unsigned dev USB boot.

|                     | Puck A (mesh1, flashed cleanly)             | Puck B (suspected "walled")                       |
|---------------------|----------------------------------------------|----------------------------------------------------|
| eMMC chip           | `4FTE4R` rev 0.1, S/N 2789407485             | `4FPD3R` rev 0.2, S/N 2986147503                  |
| vboot key block     | RSA padding check **fails**, falls through to hash-only ("In recovery mode or dev-signed kernel") | RSA signature **passes outright** — kernel preamble + partition both good, "Same kernel version" |
| rootfs mount        | `root=PARTUUID=cc24514c-...` (label-based)   | `root=/dev/dm-0` (dm-verity verified rootfs)      |
| cmdline extras      | `noinitrd`, no DRM tracing                   | `drm.trace=0x106`, `dm_verity.dev_wait=1`         |
| Kernel PARTUUID     | `cc24514c-062f-2e48-b5f6-05f5233499a4`       | `f1bcbf9a-8da7-454b-98f5-208d7cde9fb7`            |
| Vboot dev branch    | `VbBootDeveloper() - trying fixed disk`      | Multiple `VbBootDeveloper() - user pressed Ctrl+U; try USB` retries before falling back to fixed disk |
| Boot lands at       | ChromeOS Developer Console banner            | Direct kernel boot (no banner shown)              |
| coreboot bootblock  | `60d1b1c` 2017-01-09                         | `60d1b1c` 2017-01-09 (same — wall is above coreboot) |

Puck A flashed first try with the standard kkestell procedure (the rest
of this post). Puck B has not been flashed yet at time of writing. The
dm-verity rootfs + hard RSA-pass on B is exactly the "newer firmware
that refuses unsigned dev USB boot" pattern. We won't know for sure
until we try the SW7 dance with an OpenWrt USB drive on B.

## Workarounds for "walled" pucks — what's out there

Searched the OpenWrt forums, kkestell guide, MrChromebox docs, and
related write-ups for a known fix when the SW7 USB-boot reverts on a
newer-firmware puck. **There is no clean software-only fix.** Summary:

- **OnHub Recovery Utility** only offers a single firmware version (the
  latest). You can't pick an older image from inside it. So flashing
  the "factory image" doesn't downgrade past whatever the puck is
  already running.
- Older Google WiFi factory firmware images are **not publicly
  hosted** anywhere I could find. Forum threads explicitly note "such a
  file doesn't seem to be publicly available."
- **galeforce** (`github.com/marcosscriven/galeforce`) is a rooted
  Google firmware fork, but it uses the same SW7 dev-mode USB-boot
  path — so a walled puck reverts galeforce too. Confirmed in an
  earlier session of this project (see memory note 2026-05-22).
- **CH341A SPI flash programmer** + SOIC-8 clip is the hardware-level
  fallback used on Chromebooks via
  [MrChromebox's unbricking guide](https://docs.mrchromebox.tech/docs/support/unbricking/unbrick-ch341a.html).
  The MrChromebox docs are Chromebook-specific and don't mention
  AC-1304, but the principle should transfer: open the case, clip the
  Winbond SPI chip, dump current firmware, write an older coreboot
  image that allows dev USB boot. The catch: where do you get an older
  coreboot image for Gale? Nobody seems to have published one. So this
  is "dump it yourself from an older puck, then reflash a walled one."
- **SuzyQ CCD unlock** could in theory let you change CCD/firmware-mgmt
  flags from a Mac/Linux host via the existing debug interfaces — but
  on Gale's H1 chip the unlock procedure is undocumented (see
  [`docs/ccd-unlock-research.md`](ccd-unlock-research.md)). Open research.
- **Source older un-updated pucks** — buy used / pull from sat-offline
  inventory. Lottery.
- **Pragmatic exit:** if N of M pucks are walled, deploy 1 OpenWrt
  router + use cheap dedicated OpenWrt APs (e.g. GL.iNet, Belkin
  RT3200, or any IPQ40xx ≠ Gale) for mesh. Stops the puck collection
  from being load-bearing.

So the honest blog-post conclusion for a walled puck is: **try the
SW7 dance first, and if it reverts, you're looking at CH341A or
recycling.** The serial console doesn't unblock flashing on a walled
puck, but it DOES let you definitively confirm which side of the wall
each puck is on without trial-and-error LED-watching.

### Confirming "walled" empirically — the SSH race that never wins

After SW7-dancing a suspect-walled puck, the LED actually does the
"rapid blue → steady blue → purple loop" thing, and `ping 192.168.1.1`
shows a fascinating pattern: complete silence, then ~7 seconds of
successful replies (with the classic backlog-flush latency drop), then
back to silence. Repeat every ~3 minutes.

The kernel IS booting. LAN comes up. Networking works for ~7 seconds.
Then the firmware kills it.

To prove SSH never gets a chance, I race-looped `ssh root@192.168.1.1`
for 5 minutes, gated by a 200 ms ping check so we only attempt during
live windows:

```
race start: Tue May 26 19:52:59 EDT 2026
[8 ping-OK windows across 5 minutes]
ssh: connect to host 192.168.1.1 port 22: Connection refused
ssh: connect to host 192.168.1.1 port 22: Connection refused
...
race end: Tue May 26 19:58:00 EDT 2026, 135 attempts, 0 SSH successes
```

`Connection refused` (not timeout) — the puck is responding on layer 3
but dropbear has not yet bound port 22. Per `procd`'s default startup
order in OpenWrt 25.12.4, dropbear comes up after networking, and
*after* the ~7 second window the firmware's signature watchdog kills
the kernel. So:

- ping works → kernel runs, ethernet driver up, IP assigned
- TCP SYN to :22 → RST → dropbear never listened
- 7s later → puck reboots → cycle repeats

This rules out any "race the dropbear and ssh in and run cgpt to mark
the kernel successful" approach — at least at default OpenWrt boot
priority. A custom OpenWrt build that brought dropbear (or some other
sshd) up in initramfs at preinit time *might* win the race, but I
haven't built it. Even then, marking the kernel "successful" likely
won't help because earlier sessions also tried initramfs-only boot
(no rootfs dependency) and the wall reverts that too. The wall is at
the firmware's signed-kernel enforcement layer, not at boot-success
GPT flags.

So if you have the time and patience to attempt a flash on a suspected
walled puck *with* the SuzyQ also attached for diagnosis: you'll get
the same conclusion in about 5 minutes of race-looping ping+ssh,
without ever opening the case for SW7. Even without serial, the ping
pattern alone is diagnostic.

Sources I checked:
- [OpenWrt forum: Google WiFi flash firmware failure thread](https://forum.openwrt.org/t/google-wifi-flash-firmware-failure/184617)
- [OpenWrt forum: how can to stock firmware in google wifi (ac 1304)](https://forum.openwrt.org/t/how-can-to-stock-firmware-in-google-wifi-ac-1304/235366)
- [OpenWrt forum: Finally installed OpenWRT on my Google WIFI(AC-1304)](https://forum.openwrt.org/t/finally-installed-openwrt-on-my-google-wifi-ac-1304/183541)
- [MrChromebox CH341A unbricking](https://docs.mrchromebox.tech/docs/support/unbricking/unbrick-ch341a.html)
- [galeforce project](https://github.com/marcosscriven/galeforce)
- [krishnendu.com — OpenWrt on AC-1304](https://krishnendu.com/openwrt-on-google-wifi-ac1304-easy-way-thow-i-turned-my-lost-qr-google-wifi-into-an-openwrt-beast/)
- [Google Nest Community: need to downgrade my AC-1304](https://www.googlenestcommunity.com/t5/Nest-Wifi/Need-to-downgrade-my-Google-Wifi-Ac1304/m-p/408325)

## Hack attempt #1 — talking to the SPI flash chip through the SuzyQ

After confirming the walled puck is genuinely walled (5 minutes of
SSH-race, 135 attempts, 0 hits), I spent an hour cracking open the
Gale debug device's third USB interface — the one we'd been ignoring
because it was silent during boot.

Turns out **subclass `0x51` = `USB_SUBCLASS_GOOGLE_SPI`** —
`flashrom`'s "raiden" protocol. If accessible, this lets you READ AND
WRITE the puck's SPI flash chip directly through the $7 cable. No
soldering, no CH341A, no SOIC-8 clip.

I sent a JEDEC-ID SPI READ request (opcode `0x9F`) through the bridge:

```
[v1 xfer] JEDEC_ID: OUT 01039f (write_count=1, read_count=3)
  IN 05009f9f9f  status=0x0005  payload=3B
  ERROR status 0x0005
```

`status=0x0005` is the defined error code from the chromiumos
`chip/stm32/usb_spi.h` header:

> 0x0005: The SPI bridge is disabled.

So:
- ✓ Iface 3 IS the SPI bridge, on `bInterfaceProtocol=0x01` (v1 spec)
- ✓ It speaks the raiden protocol — defined error codes, no garbage
- ✓ Writes are NOT blanket-blocked (unlike the AP/EC UART writes, which
  silently time out)
- ✗ The bridge is **explicitly disabled** by the H1's CCD lock

So the hardware path is fully implemented and works. The bridge has
just been deliberately turned off. On a Chromebook you'd flip it on
with `gsctool ... ccd-set FlashAP`, but Gale doesn't expose the
`USB_SUBCLASS_GOOGLE_UPDATE` (`0x53`) interface that `gsctool` talks
to. Wedge identified, lock still in place.

This still feels like the highest-leverage research direction in the
whole project. If anyone reading the blog post knows the Gale-specific
CCD unlock — please get in touch. Notes are at
[docs/ccd-unlock-research.md](ccd-unlock-research.md) and the probe
tool is [tools/gale-spi-probe](../tools/gale-spi-probe).

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
