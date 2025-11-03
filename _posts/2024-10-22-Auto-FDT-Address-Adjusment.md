---
title: "Auto FDT Address Adjustment in U-Boot"
date: 2024-10-22
categories: [U-Boot]
tags: [fdt, u-boot]
---


## Problem
You may see this error in U-Boot when booting a kernel: 
    
`Did not find a cmdline Flattened Device Tree Could not find a valid device tree`

Root cause: the kernel image ended up larger than expected and its end address overlaps (overwrites) the DTB already placed in memory. U-Boot loads kernel and FDT according to configured addresses (variables like `fdt_addr_r`). If the kernel's end address grows and overlaps the FDT location, the kernel never sees a valid device tree.

## Quick manual fix
Move the FDT address (`fdt_addr_r`) further up memory so it lives beyond the kernel end address and won’t be overwritten. That’s fine for one-off cases, but it’s manual and brittle when kernel sizes change.

## Auto solution (what I implemented)
I created a general, boot-time auto-adjust mechanism that:

- Detects the **actual kernel size / end address** at boot.
- If the configured FDT address would be overwritten by the kernel, **computes a new safe FDT address** (linearly offset based on kernel size) and updates `fdt_addr_r` before the FDT is used.
- Works for both interactive `boot` command flows and automated `sysboot`.
- Is shipped as a bash script you place in the U-Boot root directory and enable (it’s enabled by default in my package).
- Targeted and tested for U-Boot **v2023.10** (23.10). Might work on other versions but behavior/paths can vary.

## How it behaves (conceptually)
1. At boot (or when `boot`/`sysboot` runs), U-Boot normally prepares kernel and FDT using configured addresses.  
2. The auto-adjust logic calculates the kernel end address from the loaded kernel size.  
3. If `fdt_addr_r` < kernel_end + padding, the script bumps `fdt_addr_r` upward to a safe value (kernel_end + padding).  
4. U-Boot proceeds with the adjusted FDT address; the FDT is no longer overwritten by the kernel.

## Benefits
- No manual edits to device tree address variables after each kernel change.  
- Works automatically for scripted and interactive boots.  
- Prevents hard-to-debug boot failures caused by image-size growth.

## Limitations & notes
- Default behavior in my package is **enabled**; you can disable if you prefer manual control.  
- Make sure your chosen new FDT address does not collide with other memory usages (ramdisk, initrd, etc.). The script uses a simple padding policy; adjust if your board has special constraints.  
- This is a runtime mitigation — best practice remains: choose stable, non-overlapping load addresses in your board config when possible.  
- Tested on U-Boot 23.10; if you run a different U-Boot version, test carefully in QEMU or on a dev board.

## Where to get it
- Script (bash): https://mega.nz/file/46tCmaAY#o57eQxyM_-iFh1MAGwsUMJW264ESE8LG8J2-yXlxuPI  
- Documentation for the script: https://mega.nz/file/UjlUSLiS#tnx-g-EC0YlZp1s3e2JP034hNCtvINt37Pr99K4KKl0

![Auto-fdt-addr](/assets/img/Auto-fdt-addr-adj.png)
