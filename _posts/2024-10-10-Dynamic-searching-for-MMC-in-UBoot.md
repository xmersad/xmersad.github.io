---
layout: post
title: "Dynamic searching for MMC in U-Boot (`disk/part.c`)"
date: 2025-10-09
categories: [u-boot, bootloader, mmc, embedded]
tags: [mmc, u-boot, device-tree, boot, fallback, rootfs]
excerpt: "A simple patch/technique for making U-Boot try multiple MMC device numbers (e.g., mmc1 -> mmc0) dynamically so you don't need to rebuild targets when device numbers shift."
---

# Dynamic searching for MMC

**Short summary**  
Some boards change the MMC device number after a DT or platform change (for example `mmc0` becomes `mmc1`). Rebuilding every consumer of MMC settings (sysboot, script locations, image loaders) is inconvenient. A small, pragmatic change in `disk/part.c` lets U-Boot try alternate MMC numbers at runtime (fallback search) and therefore handle those device-number shifts without rebuilding every target.

---

## Problem description
On some targets the kernel/device-tree ordering or controller probing can change the logical MMC device index exposed to U-Boot. If your boot configuration, sysboot, or scripts point to a fixed MMC number (e.g., `"1"`), and the board comes up as `"0"` (or vice versa), file reads/writes and boot attempts will fail.

Common symptoms:
- U-Boot prints "Bad device specification" for a valid partition path.
- `fatload`/`ext4load`/`load` commands fail unexpectedly after a DT or hardware change.
- Automated flashing or sysboot logic breaks when the MMC index changes.

---

## Pragmatic solution (fallback search)
Instead of hard-coding a single MMC number, modify the device lookup logic so U-Boot attempts the primary index and, on failure, tries alternate indices. The pattern below illustrates trying `"1"` first, then falling back to `"0"`. You can extend this to try more indices depending on board variants.

**Context provided:** line `536` in `v2023.10` (example location where device lookup happens).

**Illustrative snippet (C):**
```c
/* Look up the device */
dev = blk_get_device_by_str(ifname, "1", dev_desc);
printf("\n***Trying to read mmc 1***");
if (dev < 0) {
    printf(" ->Failed \n***Bad mmc number... try mmc 0*** -> ");
    dev = blk_get_device_by_str(ifname, "0", dev_desc);
    if (dev < 0) {
        printf("** Bad device specification %s %s **\n", ifname, dev_str);
        ret = dev;
        goto cleanup;
    }
}
```
**Behavior:**  
1. Try to bind to `mmc 1` first.  
2. If that fails (`dev < 0`), try `mmc 0`.  
3. If both fail, report a bad device specification and bail out.

You can extend this pattern to try additional indices (e.g., `"2"`, `"3"`), or implement a small loop to iterate a configurable list of candidate indices.

---

## Integration tips
1. **Make it configurable** — instead of hard-coding indices in source, consider reading a short list from an environment variable (for example `mmc_try="1,0"`) or from board-specific configuration. This avoids repeated source edits across boards.  
2. **Keep user-visible logs** — print which device index succeeded to make debugging easier across different hardware revisions.  
3. **Fail fast per attempt** — ensure fallback attempts fail quickly to avoid adding significant boot latency. Avoid long timeouts for each try.  
4. **Localize the change** — keep fallback logic limited to the device-lookup path so other code relying on explicit device numbers remains predictable.

---

## Implementation variants
- **Small-loop approach (recommended):** implement a loop over an array/list of candidate indices rather than nested `if` blocks. Easier to extend and maintain.  
- **Env-driven approach:** add an environment variable like `boot_mmc_try` containing a comma-separated list (e.g., `"1,0"`). Parse and try indices in order.  
- **Board-specific fix:** where possible, prefer fixing controller probe order or device-tree issues so device numbers are deterministic; fallback is a pragmatic workaround when fixing the root cause is impractical.

---

## Build & test notes
1. Edit the source file where `blk_get_device_by_str` is called (e.g., `disk/part.c` or the equivalent location in your U-Boot tree/version).  
2. Build U-Boot as usual for your board: `make <board>_defconfig && make -j$(nproc)`.  
3. Test in QEMU if supported, or use real hardware with a serial console to observe printed fallback messages.  
4. Validate file loads (e.g., `fatload mmc <dev> <addr> <file>`) succeed for both original and fallback device orders.  
5. Keep a recovery/revert plan in case the fallback loop introduces unexpected latency or interaction issues.

---

## Caveats & considerations
- This is a runtime workaround — not a replacement for correct device-tree/controller ordering. Use it when rebuilding many artifacts is impractical.  
- Overly aggressive fallback (trying many indices) can increase boot latency and may mask underlying configuration issues; prefer a short candidate list.  
- Ensure compatibility with your U-Boot version and follow project coding standards if you plan to upstream the change.

---

## Quick pseudo-implementation (loop + env idea)
1. Read env `mmc_try` (default: `"1,0"`).  
2. Split the string into tokens and iterate over them.  
3. For each token, call `blk_get_device_by_str(ifname, token, dev_desc)`; on success use that device and break.  
4. If no token succeeds, return the original failure behavior.

---

<!-- Image placeholder: the image must be the last element. Replace the path below with your image path. -->
![dynamic mmc search diagram](/path/to/your/image.png)

