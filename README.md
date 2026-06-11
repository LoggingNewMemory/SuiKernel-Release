<img width="1920" height="1080" alt="Banner" src="https://github.com/user-attachments/assets/caae8c5e-5847-4c3b-8057-c132d9957110" />

# Release Page of SuiKernel

だって僕は星だから~ <br />
Stellar Stellar

## Features
- NTSYNC support
- Anya Thermal | Spoof Thermal to 30 degree + Disable thermal zone
- Multi Target Uname Spoof (Fix AlfaGift uname detection)
- Sandevistan Boot (MaxFreq when device boots, restore to normal after 60 seconds)
- Yamada Gaming Boost (Simply a CPU Input Boost and Schedutil Rate Limit Tuning)
- Tenebrion Battery (Minfreq when screen is off)
- BBR as Default
- I/O Sched set to none
- 300 Hz timing
- SCHED MC is enabled
- ZRAM with LZ4 Algoritm
- Sparxie Swap Tuner — Tune swapiness to 30
- Ochinai Inaho Audio — Tune SCHED_FIFO + PM QoS + audioserver thread booster (Basically to minimze audio latency)
- Enforce schedutil as GOV

## Support Me
https://sociabuzz.com/kanagawa_yamada/tribe (Global) <br />
https://t.me/KLAGen2/86 (QRIS) <br />
https://www.paypal.me/KanagawaYamada (PayPal) <br />

---

## Why Sometimes Need Force Reboot?

> When you flash a new kernel, the **first boot has extra work to do** that normal boots don't. If any of that extra work takes too long, a watchdog timer fires — and the device appears dead. A force reboot puts the hardware into a known state, so the second boot completes cleanly.

### What's Different About a First Boot?

Every time you flash a new kernel, Android treats the next boot as a special event. Several systems kick in **only on that first boot**, and they all compete for time before a deadline (the watchdog timer).

**1. SELinux Policy Recompilation**
Android re-reads and recompiles its security policy on first boot. If your kernel's SELinux hooks initialize even slightly slower than the ROM expects, the `init` process can stall — waiting for a policy that isn't ready yet.

**2. Android Verified Boot (AVB) / dm-verity Re-verification**
After flashing, AVB re-checks the integrity of the `system`, `vendor`, and `boot` partitions from scratch. This is significantly slower than a normal warm verification pass. If it takes too long, the watchdog fires before `init` even gets going.

**3. `init` Early-Mount Timing**
The first stage of `init` must mount critical partitions (`system`, `vendor`, `data`) before anything else can run. If your kernel's UFS/eMMC driver takes a few extra milliseconds to become ready, `init` sits waiting. On a normal second boot, the hardware is already warmed up and responds faster.

**4. Hardware Bringup on Cold State**
On a freshly flashed device, some hardware blocks (PMIC, clock controllers, storage controllers) are in an **unknown power state**. The kernel has to probe and initialize them from scratch. After a force reboot, those components are already powered — bringup is much faster.

**5. `ueventd` and Kernel Module Loading**
Android's `ueventd` waits for the kernel to announce all hardware devices via uevents. If modules load slowly on first boot, `ueventd` can time out waiting for a device node before it appears.

### Summary

```
Flash kernel
    └─► First boot: AVB recheck + SELinux recompile + cold hardware bringup
            └─► Takes too long → watchdog fires → device appears dead

Force reboot
    └─► Second boot: hardware warm, verifications fast, watchdog not hit
            └─► Boots normally ✓
```

This is **expected behavior** after flashing a new kernel. It does not mean your kernel is broken — the device is fine on every subsequent boot.
