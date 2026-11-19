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
| **Step 3B MGLRU (`11_mglru_kernel` test → `12_full_3a3b_build`)** | **`RisingOS-9-261116-2342`** (`rc=0`, full ROM covering 3A + 3B) | Backported Multi-Gen LRU (34 commits cherry-picked from ACK `android13-5.10`: Yu Zhao's MGLRU series plus Android/Qualcomm fixes for speculative faults, swap-full anon reclaim, per-zone reclaim and CMA). `CONFIG_LRU_GEN=y` + `CONFIG_LRU_GEN_ENABLED=y` (on at boot, `lru_gen/enabled=0x0003`), `min_ttl_ms` left at 0. Device test (runtime toggle): clean kswapd anon+file reclaim, no warnings, zram/MemoryLimiter/KSU unaffected. **Flashed 2026-11-17** (dirty sideload, slot `_b`, kernel `#7`): `lru_gen/enabled=0x0003` at boot, `recomp_algorithm` `[zstd]`, writeback `loop44`, MemoryLimiter init OK, no `cgroup.*` in cmdline, KSU root OK, camera `platform_app_36`, clean dmesg, empty crash buffer |
| **Step 4 usage-aware lmkd protection (`13_freqapps_build`)** | **`RisingOS-9-261117-0834`** (`rc=0`) | frameworks/base `f028b9b6b`: `FrequentAppTracker` scores app use (decaying, 3-day half-life, persisted in `/data/system/frequent_apps.txt`). The main processes of the top N apps (`persist.sys.lmk.protect_frequent`, default 5, 0 = off) get the lowest cached adj (900), and all other cached processes get ≥ 901, so lmkd kills them first. Kernel unchanged from 2342. Built and verified on the VM (class in `services.jar`, OTA cert = releasekey). **Flashed 2026-11-17** (dirty sideload, slot `_a`): KSU, MGLRU, zstd recompression, writeback, MemoryLimiter OK, crash buffer empty |
| **Batch build: BTF + BBR + tcmd lseek + logd 8M + camera preload (`14_batch_build`)** | **`RisingOS-9-261117-2013`** (`rc=0`, ~38 min) | kernel `680a4c9bdafc` (`CONFIG_DEBUG_INFO_BTF=y`, BBR default), vendor `5ec3a0c` (`lseek` in `tcmd.policy`), mondrian `3f51eb1` (`ro.logd.size=8M`), miuicamera `8696b1c` (`ro.config.app_preload.*`, preferred-apps XML), frameworks/base `30ce5db82` (`AppPreloader`). Built and verified on the VM (`.config` BTF/BBR/KSU/LRU_GEN=y, `.BTF` section in vmlinux, props present, OTA cert = releasekey). **Flashed 2026-11-17** (dirty sideload, slot `_b`): KSU, MGLRU, zstd, BBR, BTF (`/sys/kernel/btf/vmlinux`), MemoryLimiter, AppPreloader OK. Open: `ro.logd.size` not applied (256 KiB buffers). |
| **ROM_HANDOFF items 1–6** | committed 2026-11-17, **not built yet** | Kernel: `CONFIG_DEBUG_INFO_BTF=y` (batterystats per-app CPU power via libbpf `timeInState`), `CONFIG_TCP_CONG_BBR=y` + `CONFIG_DEFAULT_BBR=y` (BIC/Westwood/H-TCP off, no new modules). Vendor fork: `lseek: 1` in `tcmd.policy` (stops the tcmd SIGSYS crash loop that reset `persist.device_config.*`). mondrian: `ro.logd.size=8M`. MiuiCamera: default preferred activity `.Camera` for `STILL_IMAGE_CAMERA_SECURE` (no chooser on the lock-screen shortcut). Camera preload: new frameworks/base `AppPreloader` + MiuiCamera opt-in props (handoff item 2). On the phone, `statsd_enable/disable_swappiness` was created so the ROM's swappiness 120 is no longer overridden by the module's 60 |
| **RAM + battery rebuild M1–M5** | committed 2026-11-19, **not built yet**. Targeted checks passed: `m services SystemUI` + mmd/logd tests (`15_m1m2_targeted`, rc=0), host `libmmd_unit_tests` 113/113, `logd-unit-tests` 43/43, `host_init_verifier` on the changed rc files, out-of-tree kernel compile of `zram.ko`/`pid.o`/`compaction.o`/`vmscan.o` | Root causes from 39 h of telemetry on build 2013. **M1 frameworks/base:** removed the screen-on `releaseMemory` SIGKILL sweep (815 kills in 39 h, camera 13×) and hardened `AMS.releaseMemory` (permission, AMS lock, `skipCamera`, kills recorded in exit-info); `FrequentAppTracker` ignores launchers/SystemUI, 8 slots, camera pinned; `AppPreloader` screen/display checks; `ZramMaintenance` no longer needs device idle (it was 11 h overdue on `IDLE`, so zram writeback never ran: `bd_stat 0 0 0`). **M2:** UFS mq-deadline + 128 KiB RA, VM dirty/watermark tuning, cpuctl uclamp, cpusets, thermal `sconfig` 0, qca6490 WLAN power save/DTIM, PASR conflict markers, mmd writeback/recompression windows, logd honours `ro.logd.size`. **M3 kernel:** Android per-process zram writeback/prefetch ioctls (5.10 backport), `ZRAM_TRACK_ENTRY_ACTIME` split (off), compaction `fast_isolate_freepages()` limit fix. **M4 kernel:** Goodix gesture irq-wake balance. MiuiCamera: `vendor_panel_info_prop` read |

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
  - **BTF + BBR (commit `680a4c9bdafc`):** `mondrian_GKI.config` adds `CONFIG_DEBUG_INFO_BTF=y` (pahole is the Lineage prebuilt `prebuilts/kernel-build-tools/.../pahole` v1.25), `CONFIG_TCP_CONG_ADVANCED=y`, `CONFIG_TCP_CONG_BBR=y`, `CONFIG_DEFAULT_BBR=y`, and disables `TCP_CONG_BIC`/`WESTWOOD`/`HTCP` (they default to `m`). Checked with `merge_config.sh` + `olddefconfig`: `DEFAULT_TCP_CONG="bbr"`, BTF=y.
  - **M3/M4 (2026-11-19, not built yet):**
    - `mm/compaction.c`: `fast_isolate_freepages()` used `min(1U, …)` instead of `max(1U, …)` for the freelist scan limit (same fix as upstream "mm/compaction: fix 'limit' in fast_isolate_freepages", ACK `b55ca5264b0c`). Baseline had 6,080 `compact_stall` with 63.5% failures.
    - ~~`mm/vmscan.c`: `compound_head()` in the `lru_gen_look_around()` fast path~~ (`265989a`): reverted in `ca84e471569e` (2026-11-19). It had no effect on 5.10 (`activate_page()` already calls `compound_head()`), and its message described a bug 5.10 doesn't have.
    - `BACKPORT: zram: split memory-tracking and ac-time tracking` (upstream `a7a0350583ba`, Sergey Senozhatsky, Linux 6.8): new `CONFIG_ZRAM_TRACK_ENTRY_ACTIME` so age-based idle marking no longer needs `DEBUG_FS`. **Left off** in `mondrian_GKI.config`: 8 bytes per slot (~16 MiB for the 8 GiB zram), and mmd already uses its untracked idle marker.
    - `zram: factor per-slot writeback out of writeback_store()` (no functional change) + `pid: export pidfd_get_pid() for modules` (zram is `=m`).
    - `ANDROID: zram: add per-process writeback/prefetch ioctls (5.10 backport)` + `CONFIG_ZRAM_ANDROID_IOCTL=y`: `ZRAM_ANDROID_IOC_GET_VERSION` (=1), `PROCESS_RANGE_WRITEBACK`, `PROCESS_PREFETCH`, based on ACK `android16-6.12` `zram_ioctl.c` (uapi header verbatim). 5.10 has no `pp_ctl`/`wb_ctl` and no compressed writeback, so slots are collected into a bitmap and written back with the sysfs per-slot helper, and prefetch reads + recompresses with a conditional store (`ZRAM_UNDER_WB`/`ZRAM_IDLE` protocol, `-ESTALE` if the slot changed). The legacy ioctl 1 (neverallowed by sepolicy) is not implemented. mmd's sepolicy already allows these three ioctls. The framework side is DeviceConfig `activity_manager/zram_writeback_enabled` (default false). Compile-tested out of tree; **not device-tested yet**.
    - `input: goodix_berlin: balance gesture-mode irq wake across suspend/resume`: resume decided by the current `gesture_type` (changed at runtime by the touch/fingerprint HAL), not by what suspend did. That caused the `Unbalanced IRQ 322 wake disable` WARNING, or a leaked wake source.
    - Not done, on purpose: compressed writeback (needs the 6.12 writeback rework); further zsmalloc/sched/power backports (no measured target). The `[timerfd]` / `alarmtimer -16` suspend aborts are AlarmManager wake alarms via epoll `EPOLLWAKEUP`, and the IPA aborts are network/VPN traffic, not kernel bugs.
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
  - **M2 (2026-11-19, not built yet):** `vendor.prop` PASR merge-conflict markers resolved (`a03578b`). UFS `mq-deadline` instead of `bfq`, 128 KiB read-ahead, VM dirty ratios, `watermark_scale_factor` 50, cpuctl uclamp per group, cpusets foreground `0-6` / restricted `0-3` (`f335f41`). Thermal `sconfig` defaults to 0 (normal) instead of 20 (`mgame`) at boot (`49ba925`). qca6490 WLAN power save, DTIM and RX wakelock (`cfb5d36`). `f335f41` had also set `swappiness` 80 and `lru_gen/min_ttl_ms` 1000. `960a54e` restores swappiness **120** (the user's choice) and drops the `min_ttl_ms` write (stays 0 until the OOM experiment).

### C. Device Tree — `device/xiaomi/mondrian`
- **Upstream base:** `https://github.com/TTahaKeles/android_device_xiaomi_mondrian` @ `40e5c52` (`lineage-23.2`)
- **Patch:** [`patches/03_device_xiaomi_mondrian.patch`](patches/03_device_xiaomi_mondrian.patch)
- **Tracked edits:**
  - `device.mk`:
    - Added `PRODUCT_VENDOR_PROPERTIES`: `mmd.zram.enabled=true`, `mmd.zram.comp_algorithm=lz4`, `mmd.zram.size=75%`, `mmd.zram.writeback.enabled=true`, `mmd.zram.writeback.device_size=1073741824`, `mmd.zram.writeback.min_free_space_mib=2048`, `mmd.zram.writeback.idle.enabled=true`, `mmd.zram.writeback.huge_idle.enabled=true`, `mmd.zram.writeback.use_nr_tags_prop=true`, `mmd.zram.writeback.nr_tags=31` (matching UFS 32-slot queue minus 1 reserved in `ufshcd.c` so `mmd_setup` avoids sysfs SELinux denials).
    - Added `PRODUCT_PRODUCT_PROPERTIES`: `ro.lmk.kill_heaviest_task=true`, `ro.lmk.psi_partial_stall_ms=100`, `ro.lmk.thrashing_limit=100`, `ro.lmk.swap_free_low_percentage=10`, `ro.lmk.kill_timeout_ms=100`, and `persist.sys.spoof.auto_activate=false` (makes built-in spoofer strictly opt-in so external `TEESimulator`/`AlwaysStrong` works).
    - Added `PRODUCT_VENDOR_PROPERTIES`: `mmd.zram.recompression.enabled=true`, `mmd.zram.recompression.algorithm=zstd` (needs the kernel's `CONFIG_ZRAM_MULTI_COMP`; turn off with `setprop mmd.zram.recompression.enabled false` + reboot).
    - Added `PRODUCT_VENDOR_PROPERTIES`: `ro.logd.size=8M` (the default 256 KiB per buffer kept only about 40 min of logs; commit `3f51eb1`).
    - Added `PRODUCT_COPY_FILES` entry installing `configs/memory-limiter-config.xml` to `/vendor/etc/memory-limiter-config.xml`.
  - `configs/memory-limiter-config.xml` *(new)*: enables Android 17 `MemoryLimiter` (per-app cgroup v2 `memory.high` / `memory.swap.max`). Limits scale with RAM (about 50% of MemTotal for visible apps, 25% for not-visible apps, 25% swap each):

    | SKU (`minimumRequiredMemTotal`) | memVisible | memNotVisible | swapVisible | swapNotVisible |
    | :--- | :--- | :--- | :--- | :--- |
    | 8 GB (6800 MB) | 3700 | 1850 | 1850 | 1850 |
    | 12 GB (9216 MB) | 5580 | 2790 | 2790 | 2790 |
    | 16 GB (14336 MB) | 7600 | 3800 | 3800 | 3800 |

    A process is killed 30 s after its anon+swap exceeds mem+swap; cached and persistent processes are not limited. Validated against `frameworks/base/services/core/xsd/memory-limiter-config/memory-limiter-config.xsd`. Runtime kill switches: `device_config put activity_manager memory_limiter_disable_limits true` / `memory_limiter_disable_kill true`.
  - `device.mk` (M2 `5988ae0`, not built yet): mmd writeback idle window 2–6 h, 2 GiB/day budget, recompression idle window 30 min–2 h.
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
  - Added `configs/preferred-apps-miuicamera.xml` → `/system/etc/preferred-apps/miuicamera.xml` (`device.mk` `PRODUCT_COPY_FILES`, commit `00a21ba`): `com.android.camera/.Camera` is the default handler for `STILL_IMAGE_CAMERA_SECURE` (match `100000`). MiuiCamera declares that intent on `.Camera` and `.AssistantCamera`; without a default the lock-screen shortcut showed the fingerprint prompt and an "Open with" chooser. Applies on clean flashes and after "Reset app preferences".
  - `device.mk` `PRODUCT_SYSTEM_PROPERTIES` (commit `8696b1c`): `ro.config.app_preload.component=com.android.camera/.PreloadReceiver`, `ro.config.app_preload.action=miui.intent.action.POPUP_UPDOWN_FAILED`, `ro.config.app_preload.min_avail_mb=1536`. MiuiCamera's own receiver starts the process without opening the camera (as MIUI does). Measured: lock-screen shortcut to first preview frame, cold 593 ms → preloaded 498 ms (−16%); cost about 58 MB PSS while cached.
  - `sepolicy/vendor/platform_app_36.te` (`4a7555c`, not built yet): `get_prop(platform_app_36, vendor_panel_info_prop)`. This was the only camera denial on build 2013. Audit: all 9 rules of the `miuicamera_sepolicy_fix` KSU module are in the shipped vendor policy, and its `seapp_contexts` overlay was not active, so the module is redundant (disabled on the phone).

### E. Framework — `frameworks/base`
- **Upstream base:** `https://github.com/RisingOS-Revived/android_frameworks_base-new` @ `6772376fb`
- **Patch:** [`patches/05_frameworks_base.patch`](patches/05_frameworks_base.patch)
- **Tracked edits:**
  - `packages/SystemUI/src/com/android/systemui/notetask/quickaffordance/NoteTaskQuickAffordanceConfig.kt`: Fixed Kotlin compile error (`runBlocking { userResolver.getUserForHandlingNoteTaking(QUICK_AFFORDANCE) }`).
  - `services/core/java/com/android/server/spoof/AxSpoofManager.java`:
    - Honors `persist.sys.spoof.auto_activate=false` (`AUTO_ACTIVATE_PROP`) so built-in spoofing stays opt-in instead of auto-activating on first Settings launch.
    - Adds `migrateToOptIn()` (one-time cleanup clearing auto-fetched revoked keybox + PIF config on dirty flashes while preserving user-imported keyboxes).
    - Clears stale `persist.sys.pif.*` props cleanly and reacts live to `settings put secure spoof_activated 0|1` without needing a reboot.
  - **Usage-aware lmkd protection** (`services/core/java/com/android/server/am/psc/FrequentAppTracker.java` new, `OomAdjuster.java`):
    - Counts app uses (main process leaving TOP, 60 s debounce) as a decaying score (half-life 3 days), persisted in `/data/system/frequent_apps.txt`.
    - The main processes of the top N apps (`persist.sys.lmk.protect_frequent`, default `5`, `0` = off, min score 2) get `oom_score_adj` 900 exclusively; all other cached processes are kept at ≥ 901, so lmkd kills them first. Protected apps stay cached (still frozen/compacted).
    - Why: exit-info on build 2342 showed lmkd killing frequently used apps (Immich, Telegram, Proton Mail, Threads, YouTube) purely by LRU position. Log tag `FrequentApps` prints the protected set when it changes.
  - **AppPreloader** (`services/core/java/com/android/server/am/AppPreloader.java` new, one line in `ActivityManagerService.systemReady`, commit `30ce5db82`):
    - Generic and off unless the device sets `ro.config.app_preload.component` + `.action` (a receiver that only starts the process). On screen-on (+1.5 s, so a direct camera launch is not raced) it sends that broadcast if the process is not running, the package is not force-stopped, MemAvailable ≥ `ro.config.app_preload.min_avail_mb` (default 1536) and the last attempt was over 30 s ago. Kill switch `persist.sys.app_preload=false`. Log tag `AppPreloader`.

  - **M1 (2026-11-19, not built yet):**
    - `7a9ceb46c`: removed `PhoneWindowManager`'s screen-on `mMemoryOpt` (`releaseMemory(900, 20, …)` 1.25 s after every wake-up, which ignored `skipCamera` and used raw `Process.killProcess`). `AMS.releaseMemory` now needs `KILL_BACKGROUND_PROCESSES`, runs under the AMS lock, skips persistent/camera processes and kills through `removeProcessLocked` (recorded in exit-info). SystemUI `MemoryUtils` only touches cached adj ≥ 900 with `skipCamera`.
    - `cc38e8f07`: `FrequentAppTracker` only counts launchable apps (launcher/SystemUI wasted 3 of 5 slots), 8 slots, camera pinned.
    - `a42986a52`: `AppPreloader` checks the screen/display state and defers the freezer for an active camera.
    - `2cf0436e4`: `ZramMaintenance` no longer requires device idle (the job sat 11 h overdue on `IDLE`, so mmd never wrote back; a forced run on 2026-11-19 wrote back immediately), and skips in power-save or thermal ≥ moderate.

### E4. AOSP system repos
- `system/core` (fork `cute-johnny/android_system_core`, upstream RisingOS-Revived `seventeen` @ `8eb905b`) → [`patches/08_system_core.patch`](patches/08_system_core.patch): `init.rc` VM dirty writeout + `watermark_scale_factor` (`2dc9e6a`). The zram0 `idle`/`writeback` owner change `root:mmd` (`7265ad0`) was reverted in `ec78fa1`.
- `system/memory/mmd` (AOSP @ `cf66bc3`, no fork): the same owner change (`d85b5ec`) was reverted in `effabbd`, so the tree equals upstream and the manifest keeps the AOSP project. Patch 09 was removed.
- `system/logging` (fork `cute-johnny/android_system_logging`, upstream LineageOS @ `336e170`) → [`patches/10_system_logging.patch`](patches/10_system_logging.patch): logd honours `ro.logd.size` / `persist.logd.size` on all devices, not only debuggable/automotive/desktop builds (`32c9539`). Fixes the 256 KiB buffers on build 2013.
- **Why the owner change was reverted:** it was made for an EACCES that does not happen. mmd already runs with supplementary group `system` (1000), and writeback worked with the stock `root:system 0220` once the job ran. It also took access away from group `system`.
- Forked and pushed 2026-11-19; `local_manifests/mondrian.xml` now points `system/core` and `system/logging` at the forks (branch `rising-mondrian`).

### E2. Kernel Devicetrees — `kernel/xiaomi/sm8450-devicetrees`
- **Upstream base:** `https://github.com/LineageOS/android_kernel_xiaomi_sm8450-devicetrees` @ `2e649a7f` (`lineage-24.0`)
- **Fork:** `cute-johnny/android_kernel_xiaomi_sm8450-devicetrees` branch `rising-mondrian` (commit `c0f46506`; the manifest now points here)
- **Patch:** [`patches/06_kernel_xiaomi_sm8450-devicetrees.patch`](patches/06_kernel_xiaomi_sm8450-devicetrees.patch)
- **Tracked edits:**
  - `qcom/cape.dtsi`: removed `cgroup.memory=nokmem,nosocket` from `/chosen/bootargs`. This was the real source of those flags in `/proc/cmdline` (the bootloader merges the DTB bootargs); `CONFIG_CMDLINE` alone did not remove them. Shipped since build `RisingOS-9-261116-0018` (`07_nokmem_build`), but committed to a fork only on 2026-11-17.

### E3. Vendor blobs — `vendor/xiaomi/sm8450-common`
- **Upstream base:** `https://github.com/TTahaKeles/proprietary_vendor_xiaomi_sm8450-common` @ `7b040bb` (`lineage-23.2`)
- **Fork:** `cute-johnny/proprietary_vendor_xiaomi_sm8450-common` branch `rising-mondrian` (commit `5ec3a0c`; the manifest now points here)
- **Patch:** [`patches/07_vendor_xiaomi_sm8450-common.patch`](patches/07_vendor_xiaomi_sm8450-common.patch)
- **Tracked edits:**
  - `proprietary/system_ext/etc/seccomp_policy/tcmd.policy`: added `lseek: 1`. `extract-files.py` in sm8450-common already has this `blob_fixup`, but the prebuilt vendor tree lacked it. tcmd died with SIGSYS right after boot; init flagged it as an updatable process crashing and `flags_health_check` reset all `persist.device_config.*` flags. Replaces the `tcmd_off` KSU module.

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
git -C ~/android/rising/kernel/xiaomi/sm8450-devicetrees apply /path/to/patches/06_kernel_xiaomi_sm8450-devicetrees.patch
git -C ~/android/rising/vendor/xiaomi/sm8450-common apply /path/to/patches/07_vendor_xiaomi_sm8450-common.patch
```

Then commit & `git push` to your personal GitHub forks (only ~18 KB of diffs!), while keeping `vendor/lineage-priv/keys/` private offline/in `~/rising-keys-backup.tar.gz`.
