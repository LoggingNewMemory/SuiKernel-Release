# Why Sometimes a Custom Kernel Needs a Force Reboot After Flashing

**Applies to:** Android 12 · Linux 5.10 kernels  
**Author:** For kernel users who want to understand the "first boot sometimes fails" behavior

---

## The Short Answer

When you flash a new kernel, the **first boot has extra work to do** that normal boots don't. If any of that extra work takes too long, a watchdog timer fires — and the device appears dead. A force reboot puts the hardware into a known state, so the second boot completes cleanly.

---

## What's Different About a First Boot?

Every time you flash a new kernel, Android treats the next boot as a special event. Several systems kick in **only on that first boot**, and they all compete for time before a deadline (the watchdog timer).

### 1. SELinux Policy Recompilation
Android re-reads and recompiles its security policy on first boot. If your kernel's SELinux hooks initialize even slightly slower than the ROM expects, the `init` process can stall — waiting for a policy that isn't ready yet.

### 2. Android Verified Boot (AVB) / dm-verity Re-verification
After flashing, AVB re-checks the integrity of the `system`, `vendor`, and `boot` partitions from scratch. This is significantly slower than a normal warm verification pass. If it takes too long, the watchdog fires before `init` even gets going.

### 3. `init` Early-Mount Timing
The first stage of `init` must mount critical partitions (`system`, `vendor`, `data`) before anything else can run. If your kernel's UFS/eMMC driver takes a few extra milliseconds to become ready, `init` sits waiting. On a normal second boot, the hardware is already warmed up and responds faster.

### 4. Hardware Bringup on Cold State
On a freshly flashed device, some hardware blocks (PMIC, clock controllers, storage controllers) are in an **unknown power state**. The kernel has to probe and initialize them from scratch. After a force reboot, those components are already powered and initialized — bringup is much faster.

### 5. `ueventd` and Kernel Module Loading
Android's `ueventd` waits for the kernel to announce all hardware devices via uevents. If modules load slowly on first boot, `ueventd` can time out waiting for a device node (like the touchscreen or storage) before it appears.

---

## Why Does the Force Reboot Fix It?

A force reboot is essentially a **warm reboot**, not a cold start:

| Condition | First Boot (after flash) | Second Boot (after force reboot) |
|---|---|---|
| Hardware power state | Unknown / cold | Known / warm |
| AVB verification | Full re-check | Fast cached check |
| SELinux policy | Recompile + relabel | Already compiled |
| Storage driver init | Slow probe | Fast resume |
| Boot time total | Longer → may hit watchdog | Shorter → completes cleanly |

The hardware has already been initialized once. The second boot skips or fast-paths most of that heavy lifting, easily beating the watchdog deadline.

---

## How to Diagnose Which Cause Is Hitting You

After a successful second boot, run these commands:

```bash
# Check for watchdog or timeout messages from the failed boot
adb shell dmesg | grep -E "(watchdog|timeout|init|verity|first_stage)"

# Check SELinux restorecon activity
adb logcat -b all -d | grep -E "(first_boot|restorecon|selinux)"
```

Check if your device saved a crash log from the failed boot (pstore/ramoops):

```bash
adb shell ls /sys/fs/pstore/
adb shell cat /sys/fs/pstore/console-ramoops-0
```

The `pstore` log survives across reboots and shows exactly what the kernel printed before the hang. This is the most useful diagnostic tool for this problem.

---

## Common Config Flags to Check

| Config Flag | What It Does | Risk |
|---|---|---|
| `CONFIG_BOOTPARAM_SOFTLOCKUP_PANIC` | Panics on any soft lockup | A slow first-boot driver triggers a hard panic instead of recovering |
| `CONFIG_DM_VERITY` | Enables dm-verity | Must match AVB version your ROM expects |
| `CONFIG_WATCHDOG_THRESH` | Watchdog timeout threshold | Too short = fires during legitimate first-boot work |

---

## Quick Things to Try

- **Check your `fstab.hardware`** — look at `first_stage_mount` entries; a wrong flag here causes `init` to hang waiting for a partition
- **Check `on first_boot` triggers** in your device `.rc` files — any heavy synchronous work here runs before the watchdog is disabled
- **If using `schedutil`** — ensure your governor initializes before the CPU frequency is needed by early userspace
- **Compare your defconfig** against the stock kernel's — a missing `CONFIG_*` for your storage driver can cause slow initialization only on cold boot

---

## Summary

```
Flash kernel
    └─► First boot: AVB recheck + SELinux recompile + cold hardware bringup
            └─► Takes too long → watchdog fires → device appears dead

Force reboot
    └─► Second boot: hardware warm, verifications fast, watchdog not hit
            └─► Boots normally ✓
```

This is **expected behavior** for first boots after flashing a new kernel. It does not mean your kernel is broken — it means the first boot is doing more work than the watchdog allows. The device is fine on every subsequent boot.

---

*Document written for kernel builders and testers using Android 12 with Linux 5.10.*
