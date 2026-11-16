# RisingOS 17 for POCO F5 Pro / Redmi K60 (`mondrian`) — Unofficial

**Maintainer:** John ([@cute-johnny](https://github.com/cute-johnny))
**Status:** UNOFFICIAL build, not affiliated with the RisingOS team.

This repo has the local manifest, the full list of changes, and patches I use to build RisingOS 17 for mondrian. The device, kernel and vendor work is **not mine**. My changes sit on top of it (see [Credits](#credits)).

## What's different from the base trees

- **KernelSU-Next** built into the kernel
- **RAM / memory:** Android 17 `mmd` ZRAM (lz4, 75%, 1 GiB writeback), ZRAM writeback in the kernel, cgroup v2 PSI + kmem accounting re-enabled, LMK + freezer tuning, swappiness 120
- **MiuiCamera** SELinux fix for Android 17 (`platform_app_36`), which stops crashes
- **Built-in spoofer is opt-in** (`persist.sys.spoof.auto_activate=false`), so external Play Integrity modules work
- SystemUI build fix and cleanup of a merge conflict in the common tree

See [CHANGES.md](CHANGES.md) for every change. The raw diffs are in [patches/](patches/).

## Source repositories

| Path | Repo (branch) | Based on |
|---|---|---|
| device/xiaomi/mondrian | [cute-johnny/android_device_xiaomi_mondrian](https://github.com/cute-johnny/android_device_xiaomi_mondrian/tree/rising-mondrian) | TTahaKeles `lineage-23.2` |
| device/xiaomi/sm8450-common | [cute-johnny/android_device_xiaomi_sm8450-common](https://github.com/cute-johnny/android_device_xiaomi_sm8450-common/tree/rising-mondrian) | crDroid `17.0-mondrian` |
| kernel/xiaomi/sm8450 | [cute-johnny/kernel_xiaomi_sm8450](https://github.com/cute-johnny/kernel_xiaomi_sm8450/tree/rising-mondrian) | TTahaKeles `bka` |
| device/xiaomi/miuicamera-mondrian | [cute-johnny/device_xiaomi_miuicamera](https://github.com/cute-johnny/device_xiaomi_miuicamera/tree/rising-mondrian) | TTahaKeles `15-mondrian` |
| frameworks/base | [cute-johnny/android_frameworks_base-new](https://github.com/cute-johnny/android_frameworks_base-new/tree/rising-mondrian) | RisingOS-Revived `seventeen` |

Unchanged repos (kernel modules, devicetrees, vendor blobs, Dolby, MIUI camera vendor, hardware/xiaomi) are used directly from upstream. See [local_manifests/mondrian.xml](local_manifests/mondrian.xml).

Kernel source (GPLv2): the `rising-mondrian` branch above plus [KernelSU-Next](https://github.com/KernelSU-Next/KernelSU-Next) @ `4c706b59`.

## Building

```bash
repo init -u https://github.com/RisingOS-Revived/android -b seventeen --git-lfs
mkdir -p .repo/local_manifests
curl -o .repo/local_manifests/mondrian.xml https://raw.githubusercontent.com/cute-johnny/risingos_mondrian/main/local_manifests/mondrian.xml
repo sync -c -j$(nproc) --force-sync --no-clone-bundle --no-tags
ln -sf ../KernelSU-Next/kernel kernel/xiaomi/sm8450/drivers/kernelsu  # if the symlink is missing
. build/envsetup.sh && riseup mondrian userdebug && rise b
```

Generate your own signing keys into `vendor/lineage-priv/keys/` first. Keys are never published here.

## Credits

- **[TTahaKeles](https://github.com/TTahaKeles)**: device tree, kernel, kernel modules, vendor blobs, Dolby, MIUI camera. This build would not exist without his work.
- **[crDroid](https://github.com/crdroidandroid)**: sm8450-common device tree
- **[LineageOS](https://github.com/LineageOS)**: hardware/xiaomi, kernel devicetrees, the base of everything
- **[RisingOS-Revived](https://github.com/RisingOS-Revived)**: the ROM
- **[KernelSU-Next](https://github.com/KernelSU-Next/KernelSU-Next)**: root
- **Xiaomi**: proprietary blobs and firmware
- Everyone who contributed to the trees above (see each repo's commit history)
