# Flashing OpenWrt onto a Google WiFi (Gale, AC-1304)

This doc combines the well-tested USB-boot procedure from
[kkestell/openwrt-on-google-wifi](https://github.com/kkestell/openwrt-on-google-wifi)
(itself based on
[papdee's OpenWrt forum post](https://forum.openwrt.org/t/finally-installed-openwrt-on-my-google-wifi-ac-1304/183541/2))
with the **SuzyQ serial-monitor angle** this repo enables. Credit for
the flashing flow goes entirely to kkestell + papdee — the value-add
here is "watch the boot on your Mac while you do it."

## Why bother with the SuzyQ if `dd` works fine?

The procedure below has half a dozen LED-watching steps ("LED turns
amber", "LED blinks purple", "press SW7 when it blinks again"). If
anything goes sideways you have no idea why — the puck just sits in a
blink loop. With `gale-sniff-all` running on the Mac you can see exactly
where the puck got to in the boot, whether vboot rejected the kernel,
whether USB enumeration of the flash drive happened, etc.

Run this in a second terminal during the whole procedure:

```sh
tools/gale-sniff-all --wait
```

It will reconnect each time you power-cycle.

## High-level steps

1. (If puck is in unknown state) Flash official factory firmware.
2. USB-boot OpenWrt from a flash drive.
3. `dd` OpenWrt factory image onto the internal eMMC.
4. Reboot, then upgrade to the sysupgrade image via LuCI.

## Hardware

- **Google WiFi puck** (AC-1304).
- **USB-C hub with PD passthrough** — Anker 341 (7-in-1) is known good.
  You need a USB-A port for the boot drive AND PD-in for power.
- **30 W+ USB-C PSU**.
- **2× USB 3.0 flash drives** (one for factory firmware, one for
  OpenWrt). LEDs on the drives are very useful for monitoring.
- **Phillips PH0 screwdriver** and a **spudger** to open the puck —
  you need access to internal switch **SW7** to trigger USB boot.
- **SuzyQ adapter** (optional but recommended — that's what this whole
  repo is about).

## 1. Flash official factory firmware

Use the [OnHub Recovery Utility](https://chromewebstore.google.com/detail/onhub-recovery-utility/fmgkgdalfapcmjnanilfcpkhkhedmpdm)
Chrome extension to write the latest Google WiFi factory image to a USB
drive.

Then on the puck:

1. Plug the USB-C hub into PD power (NOT into the puck yet).
2. Insert the factory firmware USB drive into the hub.
3. Hold the external **reset button** on the puck.
4. While still holding, plug the puck into the hub.
5. ~10 s later the LED turns **amber** — release the button.
6. Wait ~5 minutes. **Blue LED** = factory firmware re-flashed.

If you have the SuzyQ running you should see coreboot + depthcharge +
vboot output in the [`captures/full-boot.txt`](../captures/full-boot.txt)
style.

## 2. USB-boot OpenWrt

Write the latest OpenWrt **factory** image for Google WiFi
(`openwrt-XX.XX.X-ipq40xx-chromium-google_wifi-squashfs-factory.bin`,
from [openwrt.org/toh/google/wifi](https://openwrt.org/toh/google/wifi))
to a *second* USB drive — use the OnHub Recovery Utility with "Use
local image".

Now open the puck (PH0 screw on the bottom, gently pry the cover with
the spudger) so you can reach **SW7** on the PCB.

1. Insert the OpenWrt USB drive into the powered hub.
2. Hold the external reset button while plugging the puck into the hub.
3. ~10 s → LED turns amber → release reset button.
4. Press **SW7** once. LED blinks **purple**, puck reboots.
5. When the LED blinks purple **again**, press SW7 **a second time** —
   this triggers USB boot.
6. If successful, you can `ping 192.168.1.1` over ethernet to the LAN
   port.

## 3. Flash OpenWrt to internal eMMC

Once you're SSH-able into the USB-booted OpenWrt:

```sh
# from your Mac
scp -O openwrt-XX.XX.X-ipq40xx-chromium-google_wifi-squashfs-factory.bin \
    root@192.168.1.1:/tmp/

# from the puck
ssh root@192.168.1.1
# zero the secondary GPT header (otherwise the new image's GPT confuses things)
dd if=/dev/zero bs=512 seek=7634911 of=/dev/mmcblk0 count=33
# write the factory image to the start of eMMC
dd if=/tmp/openwrt-XX.XX.X-ipq40xx-chromium-google_wifi-squashfs-factory.bin \
   of=/dev/mmcblk0
reboot
```

Remove the USB drive. Puck boots OpenWrt from internal eMMC.

## 4. Upgrade to sysupgrade image

The factory image is a one-shot; for any future OpenWrt upgrade you
want the **sysupgrade** image, applied via LuCI at
`http://192.168.1.1/cgi-bin/luci` → **System → Backup/Flash Firmware →
Flash new firmware image**.

## What the SuzyQ trace shows for each step

| Step | Expected serial output | If you DON'T see it |
|------|------------------------|---------------------|
| 1: factory re-flash | coreboot bootblock, then depthcharge writing to eMMC | reset button not actually engaged early enough |
| 2: USB boot, SW7 press #1 | vboot finds USB-attached kernel | USB drive bad, OnHub Recovery didn't write properly |
| 2: SW7 press #2 | depthcharge picks `conf@7` (`google,gale-v2 qcom,ipq4019`), kernel handoff | wrong SW7 timing — re-do from step 2 |
| 3: post-`dd`, post-reboot | same depthcharge boot, but now from eMMC | `dd` wrote to wrong offset or wrong image |

## Common failure modes

- **Stuck on purple-blink loop:** SW7 timing is finicky. Power-cycle
  (unplug hub) and retry from step 2.
- **USB boot succeeds but `ssh` refuses:** OpenWrt sets up DHCP on the
  LAN port — make sure you're cabled into the LAN port and your Mac has
  pulled a DHCP lease in `192.168.1.0/24`.
- **`scp` "no matching host key" error:** OpenWrt uses a small key set;
  use `scp -O` (legacy SCP protocol, not SFTP — what kkestell's command
  does).
- **vboot in trace prints `In RSAVerify(): Padding check failed!` but
  then continues:** that's normal — it's vboot trying RSA verification,
  failing because we're booting a dev/recovery kernel, and falling back
  to hash-only verification. Boot continues.

## Credits

- [kkestell](https://github.com/kkestell/openwrt-on-google-wifi) — the
  flashing guide most of this is adapted from.
- [papdee on the OpenWrt forum](https://forum.openwrt.org/t/finally-installed-openwrt-on-my-google-wifi-ac-1304/183541/2) — original procedure.
- [OnHub Recovery Utility](https://chromewebstore.google.com/detail/onhub-recovery-utility/fmgkgdalfapcmjnanilfcpkhkhedmpdm) — Google's official tool.
