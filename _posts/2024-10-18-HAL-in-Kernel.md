---
title: "HAL and the Abstraction Layer for Register Access"
date: 2024-10-18
categories: [Device-Drivers]
tags: [HAL, driver, abstraction]
---


As hardware vendors expose low-level functions to access device registers, we use those functions to port software to a board — for example to run Linux or FreeRTOS.  
But drivers do **not** call vendor functions directly. Instead, there is a thin abstraction layer between drivers and the vendor HAL.

**Short description**  
Vendors provide HAL primitives (register read/write, `hal_i2c_write`, SPI ops, clock/reset helpers). Driver authors use a small, generic API from the abstraction layer (e.g. `i2c_transfer`) and state *what* they want done. The abstraction layer maps that intent to the board-specific HAL call that actually touches registers.

**How it feels for a driver writer**  
You don’t worry about vendor API names or register sequences — you call the generic API and handle results. The platform-specific mapping is handled elsewhere.

**Simple request flow**
- Driver: calls `i2c_transfer(request)` — expresses intent (transfer data).  
- Abstraction layer: inspects board/hardware config and converts the request.  
- HAL: executes `hal_i2c_write(...)` (or equivalent) and performs register access.  
- Result flows back up to the driver.

**Why this design**
- Reuse: same driver runs on multiple boards by swapping HAL implementations.  
- Isolation: hardware quirks and register details stay inside HAL.  
- Easier porting and maintenance: change HAL to support new board; drivers remain unchanged.

**Quick notes**
- The abstraction layer is small but essential — it’s the glue between generic driver logic and vendor specifics.  
- For complex buses (I²C, SPI, MMIO) the layer may include setup, timing adjustments, and error mapping.  
- Keep driver code intent-focused and lightweight; push board-specific code into HAL.


![HAL-in-Kernel](/assets/img/HAL-in-kernel.png)
