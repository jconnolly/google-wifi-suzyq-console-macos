# Unlocking a "walled" Google WiFi puck

**TL;DR — the wall is conquerable.** A Google WiFi (Gale, AC-1304) puck
that auto-updated to newer Google firmware refuses to USB-boot unsigned
kernels (OpenWrt). The fix takes ~10 minutes per puck with the SuzyQ
cable, a tiny screwdriver, and a USB drive.

The wall isn't a hardcoded signature check. It's just the `dev_boot_usb`
NVRAM flag being unset, *combined with* the standard `enable_dev_usb_boot`
script silently failing because of a missing `flashrom` binary on the
auto-updated firmware. Reflashing the official factory firmware restores
the working binary, and from there the standard kkestell flow works.

## The wall, defined

A walled puck symptom (matches the 2026-05-22 memory notes exactly):

1. SW7-dance with an OpenWrt USB drive succeeds at first: rapid blue
   LED (USB read), then steady blue (kernel loads).
2. ~7 seconds later: LED goes into a 20-blink purple loop, then
   breathing purple. Puck reboots.
3. `ping 192.168.1.1` works briefly during those 7 seconds (kernel +
   ethernet driver up) but `ssh` always returns `Connection refused`
   (dropbear hasn't bound port 22 yet when the firmware kills the
   kernel).
4. Even initramfs-only OpenWrt boot does the same — so it's not a
   rootfs / dm-verity issue.

## Why it happens

When the puck auto-updates over Google's normal firmware channel, the
new ChromeOS image lands with `dev_boot_usb` *unset* in vboot NVRAM.
Without that flag, depthcharge will load and start an unsigned USB
kernel for a few seconds, then the firmware's vboot watchdog tears it
back down because it can't confirm the kernel is signed. Hence the
7-second window, then the revert.

The standard fix on a Chromebook is `sudo enable_dev_usb_boot` from a
chronos shell. On the auto-updated Google WiFi firmware, that script
*runs* and prints `SUCCESS`, but the underlying `crossystem` call needs
`flashrom` to write the RW_NVRAM region in SPI, and the binary isn't
present in PATH on the auto-updated image — so the write silently fails
and the flag never persists.

Reflashing the **official factory firmware** restores the working
flashrom binary. From there `enable_dev_usb_boot` actually persists,
and you're back to the standard kkestell flow.

## Verification status (2026-05-27, A/B tested)

The recipe has been **verified end-to-end on three walled pucks** and
two of the candidate "skip me" steps have been A/B tested. Results:

| Step | Question | Test result |
|------|----------|-------------|
| **WP screw removal** | Is this strictly needed to type at UART? | **REQUIRED.** Tested on puck B with WP screw IN: both EC_PD and AP UART writes return `[Errno 60] Operation timed out`. Removing the screw immediately unlocks UART writes. Without it, you can't type `chronos` at the login prompt. |
| **Factory recovery flash** | Always needed, or only on auto-updated pucks? | **REQUIRED on auto-updated pucks.** Tested on puck B with dev mode enabled but no factory recovery: ChromeOS gets stuck in an endless dev-mode-first-boot powerwash cycle — `localhost login:` flashes briefly each cycle but Linux reboots before chronos shell stabilises, so `enable_dev_usb_boot` never gets a chance to run. After factory recovery, the powerwash completes in 3-5 min and chronos sticks. Hypothesis: the auto-updated image is missing a component (probably `flashrom`) that the powerwash flow needs to finish. |
| SW7 recovery dance for dev mode | Already-in-dev-mode pucks could skip the post-recovery dance | Untested but probably true — factory recovery wipes the TPM dev flag, so the SW7 dance must come after recovery regardless. |
| Powerwash wait time | Can we cut the 3-5 min wait? | Untested. The cycle is firmware-driven; probably can't be shortened from the host side. |
| `crossystem dev_boot_signed_only=0` + `dev_default_boot=usb` | Are these belt-and-suspenders next to `dev_boot_usb=1`? | Untested but probably yes — `dev_boot_usb=1` is the load-bearing flag per `enable_dev_usb_boot`'s own output. The other two just make USB the preferred boot source. Safe to keep; cheap to set. |
| Static IP on Mac side after eMMC boot | Or was DHCP just slow? | Untested in isolation. Static works reliably; DHCP race-with-dnsmasq-startup is well documented. Keep static fallback. |

**The recipe below is the minimum-known-working set.** Both attempts
to skip steps confirmed those steps are necessary. If you have a
*never-online* puck (no auto-update), it's possible the recovery
flash can be skipped — but I don't have one of those to test on.

## The unlock — 10 minute recipe (unverified-minimum)

### What you need

- The puck (case open, SW7 accessible)
- The SuzyQ cable (this repo's reason for existing)
- A small screwdriver (PH0)
- A USB drive (8GB+) — will be wiped twice
- ~2 GB of disk on your Mac
- A USB-C hub with PD passthrough + USB-A port (the kkestell flow's
  one)

### Steps

1. **Find and remove the puck's hardware write-protect screw.** It's a
   silver screw with a conductive washer bridging multiple PCB pads,
   usually near the SPI flash chip + H1 chip on the mainboard. The
   washer is the giveaway — case screws don't have one. Save the screw
   on a piece of tape so you can re-install later if you want HW WP
   back on.

2. **Download Google's official Gale recovery image:**

   ```sh
   curl -L -o ~/gale-recovery.zip \
     https://dl.google.com/dl/edgedl/chromeos/recovery/chromeos_9334.41.3_gale_recovery_stable-channel_mp.bin.zip
   # sha1: 3914470f0f3417cbd876c238fe495d65562c4f6e
   unzip ~/gale-recovery.zip
   # -> chromeos_9334.41.3_gale_recovery_stable-channel_mp.bin (1.84 GB)
   ```

3. **Write the recovery image to your USB drive:**

   ```sh
   diskutil unmountDisk /dev/disk4
   sudo dd if=chromeos_9334.41.3_gale_recovery_stable-channel_mp.bin \
           of=/dev/rdisk4 bs=1m conv=sync
   ```

4. **Reflash factory firmware on the puck:** hub powered, USB drive in
   hub, hold puck reset button, plug puck USB-C into hub, ~10s LED
   amber, release reset. Wait ~5 minutes for the reflash to complete
   (LED goes back to solid blue when done). This wipes the
   auto-updated firmware and re-installs a clean ChromeOS factory
   image that has working binaries.

5. **Re-enable developer mode via SW7 recovery dance:**
   - Unplug puck from hub
   - Connect the **SuzyQ** to the puck + Mac (powers the puck via USB)
   - Hold puck's reset button, then plug the SuzyQ in (puck powers on
     with reset held)
   - ~10s → LED amber → release reset
   - Press SW7 once (enters depthcharge recovery / VbBootRecovery)
   - Wait ~3s, press SW7 again — this triggers
     `VbUserConfirms(Yes)` → `SetVirtualDevMode TPM 0x00 -> 0x02` →
     `VbAllowUsbBoot` → cold reboot
   - Puck reboots into dev mode

6. **Wait** ~3-5 minutes for ChromeOS's first-boot-in-dev-mode
   powerwash to complete. The puck will cycle through depthcharge +
   the "Developer Console" banner a few times. Be patient — do NOT
   send any characters during this period, depthcharge will eat them
   as keypresses and may accidentally toggle dev mode back off (ask
   me how I know).

7. **Log into chronos over the SuzyQ AP UART** (interface 1):
   ```py
   # see tools/gale-spi-probe for the boilerplate of talking iface 1
   # send "chronos\r" — no password required on first boot
   # you should get "chronos@localhost $ "
   ```

8. **Run the magic commands:**
   ```sh
   sudo enable_dev_usb_boot
   # SUCCESS: Booting any self-signed kernel from SSD/USB/SDCard slot is enabled.
   sudo crossystem dev_boot_usb=1 dev_boot_signed_only=0 dev_default_boot=usb
   sudo crossystem dev_boot_usb   # should print 1
   ```

9. **Rewrite the USB drive with OpenWrt factory.bin** (overwriting the
   Gale recovery image we put there in step 3):
   ```sh
   diskutil unmountDisk /dev/disk4
   sudo dd if=openwrt-XX.XX.X-ipq40xx-chromium-google_wifi-squashfs-factory.bin \
           of=/dev/rdisk4 bs=1m conv=sync
   ```

10. **Standard kkestell flow** ([docs/flashing.md](flashing.md) from
    step 2 onwards): unplug SuzyQ, plug puck → hub with OpenWrt USB
    drive, SW7 dance — this time the USB-boot **doesn't get reverted**
    because `dev_boot_usb=1` persisted. Puck lands at steady blue
    running OpenWrt from RAM. `ssh root@192.168.1.1`, `scp` + `dd` the
    factory.bin onto `/dev/mmcblk0`, reboot, done.

## Optional: re-install the WP screw

After step 10, the puck is running OpenWrt and the original Google
firmware is gone. You can re-install the HW WP screw to restore
hardware write-protection on the SPI flash chip — though there's not
much point, since OpenWrt has its own `/dev/mtd` access if you wanted
to modify firmware anyway.

The unlock is "sticky" — `dev_boot_usb=1` lives in vboot NVRAM which
is a separate region from coreboot/AP firmware, and HW WP only
protects the coreboot region. So re-installing WP doesn't re-lock dev
USB boot.

## How this was discovered

I tried every software-only path first (SuzyQ SPI bridge probe,
1024-request control-transfer fuzz, GSC console hunt). All dead-ended
at "the SPI bridge is disabled" or "interface not present." See
[`docs/ccd-unlock-research.md`](ccd-unlock-research.md).

The actual unlock came from noticing that:
- HW WP screw removal *did* unlock CCD UART writes (we could type at
  the puck's AP UART console), even though it didn't unlock the SPI
  bridge.
- Typing `chronos` at a Linux getty over that UART gave us a shell
  with no password.
- That shell ran `sudo enable_dev_usb_boot` which printed `SUCCESS`,
  but the flag didn't persist because the auto-updated image was
  missing `flashrom`.
- Reflashing the official Google recovery image restored the working
  binaries, after which the same command worked correctly.

So the wall is software-policy-driven, not hardware. The interesting
finding is that the SuzyQ + a $1 screw removal is enough to defeat it.
