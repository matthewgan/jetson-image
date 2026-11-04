# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Build tooling for minimal NVIDIA Jetson SD/USB disk images: a custom arm64 Ubuntu rootfs combined with an NVIDIA L4T BSP. There is no application code, test suite, or linter — everything is Bash scripts, Containerfiles, and BSP patches driven by `just` + `podman`. Linux host only; it needs `podman`, `just`, `jq`, `qemu-user-static` (binfmt must be registered for arm64 emulation).

## Commands

```bash
just build-jetson-rootfs <20.04|22.04|24.04>          # -> ./rootfs/
just build-jetson-image -b <board> [-r <rev>] [-d SD|USB] -l <32|35|36>   # -> ./jetson.img
just build-jetson-image -h                             # list boards / arg rules
sudo just flash-jetson-image jetson.img /dev/sdX       # dd to a block device
just clean                                             # removes rootfs/, base/, jetson.img, AND all podman images (rootless + root)
```

Example: `just build-jetson-rootfs 24.04 && just build-jetson-image -b jetson-orin-nano -d SD -l 36`

The only automated check is `.github/workflows/ci.yaml`, which builds the full board matrix on every push/PR (on tags it also publishes a GitHub release). Validating a change means doing a real build; a single-board build is the "single test".

## Build pipeline (two stages)

**Stage 1 — rootfs** (`scripts/build-base-rootfs.sh`, rootless podman):
- Builds `Containerfile.rootfs.<ubuntu>` with `--arch=arm64` (qemu emulation), `podman save` as docker-dir, then untars the layers from `manifest.json` into `./rootfs/`.
- The rootfs Containerfile is where the target OS is defined: Ubuntu packages NVIDIA's debs depend on (some pulled from older Ubuntu suites, e.g. focal on 22.04, jammy on 24.04), extra tools, `tegratop`/`impala` binaries, the `resizerootfs` first-boot service, iwd WiFi config, and the default `jetson`/`jetson` user.
- WiFi credentials are baked in at this stage, so they must exist before building the rootfs. 20.04/24.04 use iwd (`config/iwd/networks/*.psk` → `/var/lib/iwd/`). 22.04 uses netplan + systemd-networkd + wpa_supplicant instead (`config/netplan/*.yaml` → `/etc/netplan/`; put credentials in the gitignored `50-wifi.yaml`, see `50-wifi.yaml.example`), because iwd never came up (D-Bus activation timeout) on a Nano running L4T 32.7.6 (kernel 4.9). The WiFi example forces `key-management: psk`: netplan's default enables WPA3/PMF (`ieee80211w=1`), which older USB WiFi drivers reject with "failed to configure IGTK". `config/systemd/10-wlan-kernel-name.link` keeps WiFi interfaces named `wlan0` (Ubuntu would otherwise name USB adapters `wlx<mac>`).

**Stage 2 — image** (`scripts/build-jetson-image.sh`, runs under `sudo podman`, privileged):
- Validates board/revision/device/l4t combos, reads `l4t_packages.txt` (non-`#` lines) into the `L4T_PACKAGES` build-arg.
- Builds `Containerfile.image.l4t<32|35|36>` (x86 Ubuntu jammy): downloads the BSP tarball (`ARG BSP=` URL), copies `./rootfs` into `Linux_for_Tegra/rootfs`, applies `patches/`, strips GUI/gpio debs, runs `apply_binaries.sh`, chroot-installs the CUDA toolkit + `L4T_PACKAGES`, and rewrites the initrd's `resolv.conf`.
- Runs the image with `.` mounted at `/jetson`; `scripts/create-jetson-image.sh` maps `JETSON_BOARD` to NVIDIA's `jetson-disk-image-creator.sh` board name and copies `jetson.img` back out.

## Things that span multiple files

- **Board/L4T compatibility** is enforced in `build-jetson-image.sh`: nano/nano-2gb → l4t 32 only (nano needs `-r 100|200|300`); xavier → 32/35; orin → 35/36 (Orin/Xavier need `-d SD|USB`). Ubuntu 24.04 is only for Orin. CI's matrix mirrors these pairings.
- **Adding a board** requires touching both `build-jetson-image.sh` (`supported_boards` + the validation `case`s) and `create-jetson-image.sh` (BSP board name). Note `jetson-orin-nano-super` currently exists only in `create-jetson-image.sh`, so `build-jetson-image.sh` rejects it.
- **Bumping a BSP** = change the `ARG BSP=` URL in `Containerfile.image.l4tXX`. The `patches/*.diff` are plain `diff` output applied with `patch` against NVIDIA's `nv-apply-debs.sh` / `jetson-disk-image-creator.sh`, so a BSP bump can break them. CUDA package names differ per L4T (`cuda-toolkit-10-2`, `cuda-toolkit-11-4`, `cuda-toolkit`). The `L4T*_BSP` env vars in `ci.yaml` are only used for artifact names and are not tied to the Containerfiles (they're currently out of sync).
- Changes to the three `Containerfile.rootfs.*` (or three `Containerfile.image.*`) usually need applying to all of them; they are near-duplicates.
- `patches/jetson-disk-image-creator.diff` swaps `losetup -P` partition devices for `kpartx` `/dev/mapper` mappings when writing partitions into the image.
