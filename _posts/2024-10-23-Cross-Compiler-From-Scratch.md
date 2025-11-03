---
title: "Toolchain From Scratch"
date: 2024-10-23
categories: [Embedded-Linux]
tags: [cross-compiler]
---


When you develop software for architectures different from the host (usually x86-64), you need a cross toolchain. There are three common approaches:

- build a toolchain **from scratch**  
- download a prebuilt toolchain  
- build with helper tools like **crosstool-ng**

This post gives a clear **summary** of the “from-scratch” approach and what each step means (not a full how-to). I also maintain an automated repo that runs these steps for **ARM64 → RPi4** so you can reproduce the process easily.

---

## Why “from scratch” is staged
Toolchain components depend on each other. If you try to build everything at once, some tools (headers, libs, linker) won’t be available and the process fails. So the build is staged into minimal usable parts that are progressively extended.

---

## High-level steps
1. **Build Binutils (base)**  
   - Build assembler, linker, objdump, etc. These are needed to assemble and link all subsequent components.

2. **Install Kernel headers**  
   - Install target kernel headers (or arch headers). These are required before building a C library or parts of GCC that target the kernel ABI.

3. **Build an initial (bootstrap) GCC**  
   - A minimal C compiler that can compile the C library. This compiler is often built without the full libstdc++/libgcc support.

4. **Build and install libc for the target**  
   - Provide the standard C library (glibc, musl, or uclibc) for the target so user-space programs can link.

5. **Build the final GCC**  
   - Rebuild GCC now that the target libc and headers are available — producing a full-featured cross-compiler.

---

## Repo (automated scripts)
I prepared scripts that automate these steps for **ARM64 → Raspberry Pi 4**. You only need to create a working directory; scripts will download, configure, build and install the toolchain there.

Repository:  
https://github.com/xmersad/Cross-Compiler-From-Scratch

---

## Notes & practical tips
- If you only need to compile **kernel-space** code (kernel, U-Boot, bare-metal firmware), you **don’t need libc** — a bare-metal/hosted cross-compiler is sufficient.  
- Building a toolchain from scratch is **educational** — in production you usually use a reliable prebuilt toolchain (they essentially automate the same steps).  
- Choose which C library to use (glibc vs musl vs uclibc) early — this affects configuration and ABI behavior.  
- Cross-compilation involves toolchain prefixes (e.g. `aarch64-linux-gnu-`) — keep the prefix consistent across build steps.  
- Use an isolated prefix directory (e.g. `/opt/cross/aarch64/`) so the host toolchain is untouched.  
- For reproducibility, pin source versions for binutils, gcc, and libc, and keep a script to rebuild from scratch.

---

## Summary
Building a cross toolchain from scratch means: **binutils → kernel headers → bootstrap GCC → libc → final GCC**. The process is staged because components rely on each other. Use the provided repo if you want a repeatable ARM64/RPi4 example.

![Crosscompiler-from-scratch](/assets/img/Cross-compiler-from-scratch.png)
