---
title: "Dynamic Searching for MMC in U-Boot"
date: 2024-10-15
categories: [U-Boot]
tags: [mmc, u-boot, patch]
---


## Summary
A common issue is that after hardware changes the MMC device number can change (e.g. `mmc0` → `mmc1`), so U-Boot scripts or operations that rely on a fixed MMC number fail. Instead of changing the device tree or rebuilding for every change, modify `disk/part.c` so U-Boot tries multiple MMC numbers dynamically.

## Problem
When MMC numbering changes, operations like `env import`, `sysboot`, file read/write, or any MMC-based actions may fail because the code expects a fixed device number.

## Simple solution
Edit `disk/part.c` (example: around line 536, v2023.10) to attempt one MMC number first and fall back to another if it fails. The example below tries `mmc 1` first, then `mmc 0`. Add more attempts (`"2"`, `"3"`, ...) as needed for your board.

```c
/* Look up the device */
/* Try mmc 1 first, then fallback to mmc 0 */
dev = blk_get_device_by_str(ifname, "1", dev_desc);
printf("\n*** Trying to read mmc 1 ***");
if (dev < 0) {
    printf(" -> Failed. Trying mmc 0 ...");
    dev = blk_get_device_by_str(ifname, "0", dev_desc);
    if (dev < 0) {
        printf(" ** Bad device specification %s %s **\n", ifname, dev_str);
        ret = dev;
        goto cleanup;
    }
}

```

## Practical steps

1. Edit `disk/part.c` and add the fallback logic shown above.  
2. Keep `printf` messages concise for visibility in serial output.  
3. Rebuild U-Boot using your cross-toolchain:  
   `make CROSS_COMPILE=arm-linux-gnueabihf-`  
4. Flash the new U-Boot or test with QEMU/hardware.  
5. Observe serial logs to ensure it dynamically detects the correct MMC.  
6. If necessary, extend the fallback to include additional MMC numbers.


## Result

This simple modification allows U-Boot to dynamically find the correct MMC device,  
eliminating the need to repeatedly edit the device tree or rebuild for each configuration change.


![Reset-password](/assets/img/dynamic_searching.png)

