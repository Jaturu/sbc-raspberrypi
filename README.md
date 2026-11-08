# sbc-raspberrypi

## About this fork

This fork exists to boot Talos Linux on the Raspberry Pi 5 from NVMe. It tracks the
upstream overlay at v0.2.2 and changes one thing: the U-Boot used by the `rpi_5`
installer.

Upstream builds a single generic Raspberry Pi U-Boot, which has no driver for the
Pi 5's BCM2712 PCIe root complex or the RP1 I/O chip, so it cannot see an NVMe drive
or the onboard ethernet. This fork builds the `rpi_5` U-Boot from the
[talos-rpi5/u-boot](https://github.com/talos-rpi5/u-boot) fork at tag
`v2026.10-rpi5-1` with `rpi_5_defconfig`, which adds those drivers and a fix for the
NVMe DMA window, and adds `nvme` to the U-Boot boot targets. The generic Pi build is
unchanged. The kernel is the stock Talos kernel, no kernel changes are involved.

Resulting boot chain on a Pi 5: EEPROM firmware, `config.txt`, this U-Boot, GRUB on the
EFI partition, Talos kernel from the BOOT partition. The Pi EEPROM boot order must
include NVMe.

### Building

GitHub Actions builds the overlay and the installer on every push to `main` and
`rpi5-nvme` and on tags matching `v*`, and can be run by hand from the Actions tab
with a `talos_version` input (default `v1.14.5`). The workflow pushes:

- the overlay image, `ghcr.io/jaturu/sbc-raspberrypi:<git describe>`
- the installer image, `ghcr.io/jaturu/talos-rpi5-installer:<talos version>`
- a raw disk image as a workflow artifact, kept for 14 days

A fork of this fork publishes under its own account name instead of `jaturu`.

Both U-Boot sources are cloned from git at pinned commits so the build is
reproducible. Image Factory cannot be used with this fork, since it cannot supply a
replacement U-Boot.

### Using

Point the Talos machine configuration at the installer image:

```yaml
machine:
  install:
    image: ghcr.io/jaturu/talos-rpi5-installer:v1.14.5
```

The raw image can be written to a disk as usual. On a board that already runs another
Linux from NVMe, Talos can instead be tested from RAM and then installed over the
network, with no SD card, USB stick or physical access.

### Testing from RAM with tryboot

The Pi firmware has a one shot `tryboot` mode: on the next boot it reads
`tryboot.txt` instead of `config.txt` and clears the flag, so any further reset
returns to the existing system. Combined with `os_prefix`, a complete Talos boot can
live in a subdirectory of the existing boot partition.

1. Take `u-boot.bin`, the `bcm2712*.dtb` files and the `overlays/` directory from the
   EFI partition of the raw image, and the kernel and initramfs from the installer
   image (`usr/install/arm64/vmlinuz.efi` is a UKI, the kernel is its `.linux` section
   and the initramfs its `.initrd` section). Put them in a `talos/` directory.
2. Write `extlinux/extlinux.conf` for U-Boot, with the kernel command line from the
   UKI's `.cmdline` section plus a static address, since maintenance mode would
   otherwise use DHCP:

       DEFAULT talos
       LABEL talos
         KERNEL /talos/Image
         INITRD /talos/initramfs.zst
         APPEND talos.platform=metal ... net.ifnames=0 ip=192.168.1.50::192.168.1.1:255.255.255.0:node1:eth0:off:192.168.1.1

3. Write `tryboot.txt` so the firmware loads everything from that directory:

       os_prefix=talos/
       kernel=u-boot.bin
       arm_64bit=1

4. Copy `talos/`, `extlinux/` and `tryboot.txt` to the mounted boot partition, set the
   flag and reboot:

       sudo vcmailbox 0x00038064 4 4 1
       sudo vcmailbox 0x00030064 4 4 0   # last word must read 1
       sudo reboot

   `reboot '0 tryboot'` is supposed to set the same flag through the kernel but did
   not on Debian 13, the mailbox call is reliable.

5. Talos comes up in maintenance mode on the configured address. Check it with
   `talosctl get disks --insecure` and `talosctl get links --insecure`. Nothing on the
   disk has changed at this point and a plain reset returns to the old system. To
   install, `talosctl apply-config --insecure` with a machine configuration that names
   the installer image and the NVMe disk. The installer wipes the disk and reboots into
   Talos.

### Status

Built and installed on Raspberry Pi 5 boards (D0 stepping, 16 GB) with NVMe drives,
Talos v1.14.5, Linux 6.18. The overlay stays on the 0.2.x line while Talos ships Linux
6.18. Upstream 0.3.x device trees target Linux 6.19 and are not used here.

## Upstream README

This repo provides the overlay for building Raspberry Pi generic (models before Pi 5),
Raspberry Pi 5, and RevolutionPi Talos images.

For example usage instructions follow the boot assets guide:

* using [Image Factory](https://docs.siderolabs.com/talos/v1.13/platform-specific-installations/boot-assets#example-raspberry-pi-generic-with-image-factory)
* using [imager](https://docs.siderolabs.com/talos/latest/platform-specific-installations/boot-assets#example-raspberry-pi-overlay-with-imager)

## Overlay Options

| Option            | Description                                                         |
| ----------------- | ------------------------------------------------------------------- |
| `configTxt`       | Completely replace the `config.txt` file with the contents provided |
| `configTxtAppend` | Append the contents provided to the `config.txt` file               |
