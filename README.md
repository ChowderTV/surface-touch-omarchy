# surface-touch-omarchy

Touchscreen and Surface Pen support for the **Surface Pro 7** on a **stock Arch / Omarchy kernel**, without switching to the [linux-surface](https://github.com/linux-surface/linux-surface) kernel.

Normally, Surface touch on Linux needs the linux-surface kernel, because the touch driver (IPTS) isn't in the mainline kernel. This package builds that same driver as a DKMS module instead. It also adds a small boot service that does at runtime what the linux-surface kernel patches do at build time. Your distro kernel stays as it is.

> **Status:** works on one machine. It has been tested only on a Surface Pro 7 (SKU 1866) with `linux-omarchy` 7.2.5. Reports from other setups are welcome. Please open an issue either way.

## What works

- Multi-touch finger input
- Surface Pen: inking, pressure, hover, side button
- Survives reboots, and DKMS rebuilds the module automatically on kernel updates

Not yet tested: suspend/resume, other Surface models.

**Known issue:** pen lines sometimes break up mid-stroke. Debug captures show the pen's signal dropping out while the tip is still on the glass, which points to a weak pen battery. That hasn't been confirmed yet. See [Pen troubleshooting](#pen-troubleshooting).

## Requirements

- A Surface whose touch controller is Intel `8086:34e4` at PCI `00:16.4`. Check with:
  ```
  lspci -nn -s 00:16.4
  ```
  If that prints nothing, or shows a different ID, this package won't work on your device as it is.
- Headers for the kernel you run (Omarchy: `linux-omarchy-headers`; stock Arch: `linux-headers`).
- An Arch-based distro.

## Install

```sh
sudo pacman -S --needed base-devel git meson linux-omarchy-headers   # or linux-headers
git clone https://github.com/ChowderTV/surface-touch-omarchy
cd surface-touch-omarchy
makepkg -si
reboot
```

The build downloads iptsd's source and pinned dependencies, plus the IPTS driver from the linux-surface repo, pinned to a fixed commit and checked by checksum.

After the reboot, check that it's running:

```sh
systemctl status surface-touch 'iptsd@*'
```

## Uninstall

```sh
sudo pacman -R surface-touch-omarchy
reboot
```

## How it works

| Piece | What it does |
|---|---|
| `ipts` (DKMS, installed to `/usr/src/ipts-6.19`) | The IPTS kernel driver, taken unchanged from linux-surface's `patches/6.19/0005-ipts.patch` |
| `iptsd` 3.1.0 | The userspace daemon that turns raw touch/pen data into input events |
| `surface-touch.service` → `/usr/lib/surface-touch/setup` | Runs once at boot (details below) |

The linux-surface kernel patches change two things in the kernel that we can't change from outside it. The setup script gets the same result at runtime instead:

1. **mei_me doesn't know the touch chip.** Mainline `mei_me` has no entry for `8086:34e4`. The script adds the ID through `/sys/bus/pci/drivers/mei_me/new_id` with config index `9` (`MEI_ME_PCH12_CFG`, the config linux-surface uses).
2. **The IOMMU blocks the touch chip's memory access.** linux-surface adds a kernel quirk for this. The script instead switches only the touch chip's IOMMU group to `identity` (passthrough) mode, so you don't need to turn off the IOMMU globally with `intel_iommu=off`.

Then it loads `ipts`, and iptsd's udev rule starts the daemon.

## Caveats (please read)

- **Kernel updates could break it.** The `9` in step 1 is a position in an internal kernel list (`mei_cfg_list` in `drivers/misc/mei/pci-me.c`). If a future kernel reorders that list, touch could stop working or misbehave. If touch breaks after a kernel update, check that first.
- **Less isolation for one device.** Passthrough mode means the IOMMU no longer restricts DMA from the devices in that group (the touch chip and the main Intel ME). It's narrower than `intel_iommu=off`, but it's still a security trade-off you should know about.
- **The driver is tied to linux-surface's 6.19 patch.** If a future kernel changes an API the driver uses, the DKMS build will fail. You'd then need to update `_ls_commit` to a newer linux-surface patch.

## Pen troubleshooting

- **Pen does nothing or lines break up:** replace the pen's **AAAA battery** first. Bluetooth pairing only powers the top button. Inking runs on the battery, and a weak one makes the pen stop transmitting mid-stroke.
- **Lines still break with a good battery:** iptsd may be dropping contact because the pressure signal is near its threshold. Try lowering it:
  ```sh
  sudo surface-pen-tune 3000     # iptsd default is 10000
  ```
  This writes `/etc/iptsd.d/50-pen-sensitivity.conf` and restarts iptsd. If the pen starts inking while it's only hovering, raise the number. An example config is in `/usr/share/doc/surface-touch-omarchy/`.
- To capture raw device data for a bug report: `sudo iptsd-dump /dev/hidrawN` (find N with `systemctl list-units 'iptsd@*'`)

## Credits

All the hard work is by the [linux-surface](https://github.com/linux-surface) project: the IPTS driver and [iptsd](https://github.com/linux-surface/iptsd). This repo only packages them so they run on an unpatched kernel. Please report bugs in this packaging here, not to linux-surface.

## License

GPL-2.0-or-later, the same license as the IPTS driver and iptsd.
