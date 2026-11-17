# All Changes Made From the Original RisingOS 17 (`mondrian`) Build

**Snapshot Captured:** `2026-11-16` from the build tree (`~/android/rising/`).
**Latest Built Artifact on VM:** `RisingOS-9-261116-0018-GAPPS-UNOFFICIAL-mondrian.zip` (`3,813,063,094` bytes, SHA256 `ccbe4374bde7941dde036b65c346b58e92744c260dd63763936485a79cc5123b`).

Every modified repository on the VM is tracked in Git (`repo`), so the entire customization delta across the ~180 GB AOSP/RisingOS tree is under **18 KB of text diffs** (saved verbatim in [`patches/`](patches/)).

---

## 1. Chronological Build History (From Original Sync to Now)

| Stage | Build Log / Artifact | Summary of Changes |
| :--- | :--- | :--- |
| **Baseline (`03_build`)** | `RisingOS-9-261115-0426` | Stock RisingOS 17 sync + `03_configure_and_build.sh` config (`lineage_mondrian.mk`), private signing keys (`vendor/lineage-priv/keys/`), and `NoteTaskQuickAffordanceConfig.kt` SystemUI build fix |
| **KernelSU-Next (`04_ksu_kernel`)** | `boot.img` (`rc=0`) | Cloned `KernelSU-Next` (`4c706b59`), linked `drivers/kernelsu -> ../KernelSU-Next/kernel`, wired `drivers/Kconfig` + `drivers/Makefile`, enabled `CONFIG_KSU=y` in `mondrian_GKI.config` |
| **Phase 1 RAM (`05_ram_opt_build`)** | `RisingOS-9-261115-1939` (`rc=0`) | Enabled `CONFIG_ZRAM_WRITEBACK=y`, handed ZRAM init from `init.kernel.post_boot.sh` to Android 17 `mmd` (`lz4`, 75% size, 1 GiB writeback), raised `swappiness=120`, tuned `ro.lmk.*`, halved cached app freezer debounce to `5000ms` |
| **Issues + Step 1 (`06_issues_step1_build`)** | `RisingOS-9-261115-2250` (`rc=0`) | Fixed MiuiCamera `platform_app_36` SELinux crash, made RisingOS spoofer opt-in (`AxSpoofManager.java` + `persist.sys.spoof.auto_activate=false`), set UFS `nr_tags=31` + `ro.lmk.kill_timeout_ms=100`, cleaned upstream conflict markers in `product.prop` |
| **Cgroup v2 / `nokmem` Fix (`07_nokmem_build`)** | **`RisingOS-9-261116-0018`** (`rc=0`) | Removed `cgroup_disable=pressure cgroup.memory=nokmem` via `CONFIG_CMDLINE` in `mondrian_GKI.config` and stripped legacy cgroup v1 `memory/bg/memory.swappiness` blocks from `init.qcom.rc` & `init.qti.kernel.rc` |
| **Step 2 MemoryLimiter (`08_memlimiter_build`)** | **`RisingOS-9-261116-1850`** (`rc=0`) | Shipped `/vendor/etc/memory-limiter-config.xml` (per-app cgroup v2 limits for 8/12/16 GB SKUs, about 50% / 25% of RAM for visible / not-visible apps). `pmgd` intentionally left without a vendor config (idle) |
| **Step 3A zram recompression (`09_zram_recomp`)** | fastboot test images (shipped in full ROM `RisingOS-9-261116-2342`) | Backported the Linux 6.2 zram multi-compression series (secondary `zstd` recompression of idle/huge pages) to 5.10, `CONFIG_ZRAM_MULTI_COMP=y`, and enabled `mmd.zram.recompression.*` props. Device test: zram `mem_used` 1697 → 1302 MB (−23%) after idle recompression, no regressions |
| **Step 3B MGLRU (`11_mglru_kernel` test → `12_full_3a3b_build`)** | **`RisingOS-9-261116-2342`** (`rc=0`, full ROM covering 3A + 3B) | Backported Multi-Gen LRU (34 commits cherry-picked from ACK `android13-5.10`: Yu Zhao's MGLRU series plus Android/Qualcomm fixes for speculative faults, swap-full anon reclaim, per-zone reclaim and CMA). `CONFIG_LRU_GEN=y` + `CONFIG_LRU_GEN_ENABLED=y` (on at boot, `lru_gen/enabled=0x0003`), `min_ttl_ms` left at 0. Device test (runtime toggle): clean kswapd anon+file reclaim, no warnings, zram/MemoryLimiter/KSU unaffected |

---

## 2. Complete Per-Repo Diffs (Remote `~/android/rising/`)

### A. Kernel — `kernel/xiaomi/sm8450`
- **Upstream base:** `https://github.com/TTahaKeles/kernel_xiaomi_sm8450` @ `0c507c30a49b` (`bka`)
- **Patch:** [`patches/01_kernel_xiaomi_sm8450.patch`](patches/01_kernel_xiaomi_sm8450.patch)
- **Tracked edits:**
  - `arch/arm64/configs/vendor/mondrian_GKI.config`:
    - Added `CONFIG_KSU=y` (KernelSU-Next built in).
    - Added `CONFIG_ZRAM_WRITEBACK=y` (enables ZRAM backing device writeback sysfs for `mmd`).
    - Added `CONFIG_CMDLINE="stack_depot_disable=on kasan.stacktrace=off kvm-arm.mode=protected"` (overrides `gki_defconfig` to remove `cgroup_disable=pressure` and `cgroup.memory=nokmem` so cgroup v2 PSI + kmem accounting stay active).
  - `drivers/Kconfig`: Added `source "drivers/kernelsu/Kconfig"`.
  - `drivers/Makefile`: Added `obj-$(CONFIG_KSU) += kernelsu/`.
  - **zram multi-compression backport (Step 3A):** 17 upstream zram patches by Sergey Senozhatsky (and co-authors) from Linux 6.1/6.2: the multi-comp/recompression series plus prerequisites (huge_idle writeback, aged idle tracking, flag bits), `lib/cmdline: Export next_arg()`, a `[backport]` fix for 5.10 `zs_malloc()` returning 0 on failure (+`__GFP_CMA`), and `CONFIG_ZRAM_MULTI_COMP=y` in `mondrian_GKI.config`. Each patch is a separate commit that keeps upstream authorship.
  - **Multi-Gen LRU backport (Step 3B):** 34 commits cherry-picked from the Android Common Kernel `android13-5.10` branch, keeping upstream authorship: Yu Zhao (22: LRU list refactors, page-flags layout, MGLRU groundwork → minimal implementation → rmap locality → page-table walks → memcg optimization → kill switch → thrashing prevention → debugfs, plus later fixes), Kalesh Singh (9 ANDROID/FROMGIT fixes: speculative faults, GKI ABI fixup, anon reclaim on low/full swap, workingset accounting, per-zone reclaim, `inc_min_seq()` race, `can_swap`), Sebastian Andrzej Siewior (`lru_gen_add_mm()` IRQ-off fix), Zhaoyang Huang + Charan Teja Kalla (skip ineligible CMA pages). `mondrian_GKI.config`: `CONFIG_LRU_GEN=y`, `CONFIG_LRU_GEN_ENABLED=y` (`CONFIG_LRU_GEN_STATS` off; debugfs is disabled in this kernel anyway). Branch `rising-mondrian-mglru` was fast-forwarded into `rising-mondrian` (head `0e13448ef8a1`).
- **Untracked additions:**
  - `KernelSU-Next/` (cloned from `https://github.com/KernelSU-Next/KernelSU-Next` @ `4c706b59`)
  - `drivers/kernelsu` (symlink -> `../KernelSU-Next/kernel`)

### B. Common Device Tree — `device/xiaomi/sm8450-common`
- **Upstream base:** `https://github.com/crdroidandroid/android_device_xiaomi_sm8450-common` @ `373df2d` (`17.0-mondrian`)
- **Patch:** [`patches/02_device_xiaomi_sm8450-common.patch`](patches/02_device_xiaomi_sm8450-common.patch)
- **Tracked edits:**
  - `properties/product.prop`: Removed leftover upstream merge conflict markers (`<<<<<<< HEAD` / `=======` / `>>>>>>>`) around stale Bluetooth profile properties moved to `system_ext`.
  - `rootdir/bin/init.kernel.post_boot.sh`:
    - Replaced `configure_zram_parameters` call with comment so Android 17 `mmd` (`mmd.zram.enabled=true`) owns `/dev/block/zram0` initialization without races.
    - Changed `echo 100 > /proc/sys/vm/swappiness` -> `echo 120 > /proc/sys/vm/swappiness`.
  - `rootdir/etc/init.qcom.rc` & `rootdir/etc/init.qti.kernel.rc`:
    - Removed legacy cgroup v1 blocks (`mkdir /sys/fs/cgroup/memory/bg`, `memory.swappiness 140`, `memory.move_charge_at_immigrate 1`, `tasks` chown/chmod).

### C. Device Tree — `device/xiaomi/mondrian`
- **Upstream base:** `https://github.com/TTahaKeles/android_device_xiaomi_mondrian` @ `40e5c52` (`lineage-23.2`)
- **Patch:** [`patches/03_device_xiaomi_mondrian.patch`](patches/03_device_xiaomi_mondrian.patch)
- **Tracked edits:**
  - `device.mk`:
    - Added `PRODUCT_VENDOR_PROPERTIES`: `mmd.zram.enabled=true`, `mmd.zram.comp_algorithm=lz4`, `mmd.zram.size=75%`, `mmd.zram.writeback.enabled=true`, `mmd.zram.writeback.device_size=1073741824`, `mmd.zram.writeback.min_free_space_mib=2048`, `mmd.zram.writeback.idle.enabled=true`, `mmd.zram.writeback.huge_idle.enabled=true`, `mmd.zram.writeback.use_nr_tags_prop=true`, `mmd.zram.writeback.nr_tags=31` (matching UFS 32-slot queue minus 1 reserved in `ufshcd.c` so `mmd_setup` avoids sysfs SELinux denials).
    - Added `PRODUCT_PRODUCT_PROPERTIES`: `ro.lmk.kill_heaviest_task=true`, `ro.lmk.psi_partial_stall_ms=100`, `ro.lmk.thrashing_limit=100`, `ro.lmk.swap_free_low_percentage=10`, `ro.lmk.kill_timeout_ms=100`, and `persist.sys.spoof.auto_activate=false` (makes built-in spoofer strictly opt-in so external `TEESimulator`/`AlwaysStrong` works).
    - Added `PRODUCT_VENDOR_PROPERTIES`: `mmd.zram.recompression.enabled=true`, `mmd.zram.recompression.algorithm=zstd` (needs the kernel's `CONFIG_ZRAM_MULTI_COMP`; turn off with `setprop mmd.zram.recompression.enabled false` + reboot).
    - Added `PRODUCT_COPY_FILES` entry installing `configs/memory-limiter-config.xml` to `/vendor/etc/memory-limiter-config.xml`.
  - `configs/memory-limiter-config.xml` *(new)*: enables Android 17 `MemoryLimiter` (per-app cgroup v2 `memory.high` / `memory.swap.max`). Limits scale with RAM (about 50% of MemTotal for visible apps, 25% for not-visible apps, 25% swap each):

    | SKU (`minimumRequiredMemTotal`) | memVisible | memNotVisible | swapVisible | swapNotVisible |
    | :--- | :--- | :--- | :--- | :--- |
    | 8 GB (6800 MB) | 3700 | 1850 | 1850 | 1850 |
    | 12 GB (9216 MB) | 5580 | 2790 | 2790 | 2790 |
    | 16 GB (14336 MB) | 7600 | 3800 | 3800 | 3800 |

    A process is killed 30 s after its anon+swap exceeds mem+swap; cached and persistent processes are not limited. Validated against `frameworks/base/services/core/xsd/memory-limiter-config/memory-limiter-config.xsd`. Runtime kill switches: `device_config put activity_manager memory_limiter_disable_limits true` / `memory_limiter_disable_kill true`.
  - `overlay/Frameworks/res/values/config.xml`:
    - Added `<integer name="config_defaultFreezerDebounceTimeout">5000</integer>` (halves cached freezer debounce from 10s to 5s).
  - `lineage_mondrian.mk` *(regenerated automatically on builds by `scripts/vm/03_configure_and_build.sh`)*:
    - Sets `RISING_MAINTAINER := John`, `WITH_GMS := true`, `-include vendor/lineage-priv/keys/keys.mk`, `RisingChipset="Snapdragon 8+ Gen 1"`, `RisingMaintainer="John"`.

### D. MiuiCamera — `device/xiaomi/miuicamera-mondrian`
- **Upstream base:** `https://github.com/TTahaKeles/device_xiaomi_miuicamera` @ `5536bab` (`15-mondrian`)
- **Patch & new file:** [`patches/04_device_xiaomi_miuicamera-mondrian.patch`](patches/04_device_xiaomi_miuicamera-mondrian.patch), [`patches/04b_platform_app_36.te`](patches/04b_platform_app_36.te)
- **Tracked edits + untracked file:**
  - `sepolicy/vendor/hal_camera_default.te`: Added `binder_call(hal_camera_default, platform_app_36)`.
  - Added `sepolicy/vendor/platform_app_36.te`: Mirrors `platform_app.te` permissions for Android 17's `platform_app_36` domain (`MiuiCamera` targets SDK 34 < 37), preventing `SIGSEGV` in `libmicampostproc_client.so`.

### E. Framework — `frameworks/base`
- **Upstream base:** `https://github.com/RisingOS-Revived/android_frameworks_base-new` @ `6772376fb`
- **Patch:** [`patches/05_frameworks_base.patch`](patches/05_frameworks_base.patch)
- **Tracked edits:**
  - `packages/SystemUI/src/com/android/systemui/notetask/quickaffordance/NoteTaskQuickAffordanceConfig.kt`: Fixed Kotlin compile error (`runBlocking { userResolver.getUserForHandlingNoteTaking(QUICK_AFFORDANCE) }`).
  - `services/core/java/com/android/server/spoof/AxSpoofManager.java`:
    - Honors `persist.sys.spoof.auto_activate=false` (`AUTO_ACTIVATE_PROP`) so built-in spoofing stays opt-in instead of auto-activating on first Settings launch.
    - Adds `migrateToOptIn()` (one-time cleanup clearing auto-fetched revoked keybox + PIF config on dirty flashes while preserving user-imported keyboxes).
    - Clears stale `persist.sys.pif.*` props cleanly and reacts live to `settings put secure spoof_activated 0|1` without needing a reboot.

### E2. Kernel Devicetrees — `kernel/xiaomi/sm8450-devicetrees`
- **Upstream base:** `https://github.com/LineageOS/android_kernel_xiaomi_sm8450-devicetrees` @ `2e649a7f` (`lineage-24.0`)
- **Fork:** `cute-johnny/android_kernel_xiaomi_sm8450-devicetrees` branch `rising-mondrian` (commit `c0f46506`; the manifest now points here)
- **Patch:** [`patches/06_kernel_xiaomi_sm8450-devicetrees.patch`](patches/06_kernel_xiaomi_sm8450-devicetrees.patch)
- **Tracked edits:**
  - `qcom/cape.dtsi`: removed `cgroup.memory=nokmem,nosocket` from `/chosen/bootargs`. This was the real source of those flags in `/proc/cmdline` (the bootloader merges the DTB bootargs); `CONFIG_CMDLINE` alone did not remove them. Shipped since build `RisingOS-9-261116-0018` (`07_nokmem_build`), but committed to a fork only on 2026-11-17.

### F. Build Side-Effect / Private Keys (Do Not Push to Public Forks)
- `vendor/official_devices/OTA/device/GAPPS/mondrian.json` (not published): Automatically overwritten during `bacon` OTA JSON generation.
- `vendor/lineage-priv/keys/`: Generated private release keys (`releasekey.pk8`, `platform.pk8`, `keys.mk`, backed up as `~/rising-keys-backup.tar.gz`) — keep **private**.

---

## 3. How to Replay These Exact Changes onto a Fresh Fork / Sync

If you fork the 5 upstream repos (`kernel/xiaomi/sm8450`, `device/xiaomi/sm8450-common`, `device/xiaomi/mondrian`, `device/xiaomi/miuicamera-mondrian`, `frameworks/base`) on GitHub and want to apply everything cleanly:

```bash
# 1. Kernel
cd ~/android/rising/kernel/xiaomi/sm8450
git clone https://github.com/KernelSU-Next/KernelSU-Next.git
ln -s ../KernelSU-Next/kernel drivers/kernelsu
git apply /path/to/patches/01_kernel_xiaomi_sm8450.patch

# 2. sm8450-common, mondrian, miuicamera-mondrian, frameworks/base
git -C ~/android/rising/device/xiaomi/sm8450-common apply /path/to/patches/02_device_xiaomi_sm8450-common.patch
git -C ~/android/rising/device/xiaomi/mondrian apply /path/to/patches/03_device_xiaomi_mondrian.patch
git -C ~/android/rising/device/xiaomi/miuicamera-mondrian apply /path/to/patches/04_device_xiaomi_miuicamera-mondrian.patch
cp /path/to/patches/04b_platform_app_36.te ~/android/rising/device/xiaomi/miuicamera-mondrian/sepolicy/vendor/platform_app_36.te
git -C ~/android/rising/frameworks/base apply /path/to/patches/05_frameworks_base.patch
```

Then commit & `git push` to your personal GitHub forks (only ~18 KB of diffs!), while keeping `vendor/lineage-priv/keys/` private offline/in `~/rising-keys-backup.tar.gz`.
