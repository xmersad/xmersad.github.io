---
layout: post
title: "Showing a Custom Banner in U-Boot"
date: 2025-10-09
categories: [u-boot, bootloader, build]
tags: [banner, u-boot, kconfig, build-system]
excerpt: "How to generate and embed a custom ASCII/banner graphic in U-Boot and invoke it from board code (show_banner() integration and Kconfig/Makefile flow)."
---

# Showing a Custom Banner in U-Boot

**Short summary**  
This post describes a simple workflow to convert a text string into an ASCII/banner binary (or C source), include it in the U-Boot build, and call `show_banner()` from board code (example: `common/board_f.c`) so the banner prints at boot. It also documents a Kconfig + Makefile method to pass banner text at build time.

---

## 1. Overview & tool
- Convert your desired banner text into a C-friendly form (a small library/source that renders the banner).  
- The conversion tool I used is linked here (provided by the author):  
  `https://mega.nz/file/susXiLCb#iJd5idD9uUg6K5ZpKTyYKZDEMEqqmVUPnIoVWnRPZVg`  
  (download the tool, run it on your build host to produce the banner source/header).

The tool produces a small source unit you can include in U-Boot. It exposes a function named `show_banner()` which prints the banner when called.

---

## 2. Integration points — where to call the banner
- Include the generated banner header in the C file where you want the banner to appear. For example, to print the banner after build-time messages and before DRAM information, add the include and the call in `common/board_f.c` (example insertion point around line 204):

```c
/* include the generated banner header */
#include "banner.h"

static int announce_dram_init(void)
{
    /* Print the custom banner */
    show_banner();

    puts("\nDRAM: ");
    return 0;
}
```

- Adjust the include name to match the header the converter produces (`banner.h` or `banner` depending on the tool output). Ensure the generated source is compiled and linked into U-Boot so the symbol `show_banner` is available.

---

## 3. Build-time text delivery: hardcode vs Kconfig
Two common ways to provide the banner text to the converter / build system:

### A. Hardcode
- Convert a fixed text into the generated source and add it to the repository. Simple and stable for fixed banners.

### B. Kconfig-driven (recommended for configurability)
- Add a Kconfig entry so the banner text can be set by the U-Boot configuration menu:

```text
menu "Custom Configuration"

config BANNER_TEXT
    string "Banner Text"
    default ""

endmenu
```

- Pass the selected value to the compiler as a macro so generated/compiled code can embed or react to it. Example addition in `common/Makefile`:

```makefile
CFLAGS_board_f.o += -DBANNERTEXT="$(CONFIG_BANNER_TEXT)"
```

Notes:
- The build-time macro method expects either a) your banner generator to read `BANNERTEXT` (for example a small generator step that turns the macro into C source), or b) the generated banner source to reference `BANNERTEXT`. Choose what fits your generator workflow.
- If the banner text contains spaces or special characters you may need to adjust quoting/escaping in the Makefile or use a pre-build script to generate safe source files.

---

## 4. Recommended integration workflow (step-by-step)
1. **Download** the banner converter tool to your build host (link above).  
2. **Run the tool** with the desired text to produce `banner.c` and `banner.h` (or similar outputs). Optionally automate this step in the project build if you use Kconfig.  
3. **Add the generated source** to your U-Boot build tree (e.g., place `banner.c` under `common/` or a board-specific folder). Ensure the build system compiles it (update `Makefile` or SRCS list if needed).  
4. **Include the header** (`#include "banner.h"`) in the target C file(s) and call `show_banner()` at the desired boot point (for example, in `announce_dram_init()` as shown above).  
5. **(Optional) Hook Kconfig**: add `config BANNER_TEXT` and pass it as a macro via `CFLAGS` or implement a pre-build step that generates banner source from the `CONFIG_BANNER_TEXT` value.  
6. **Build U-Boot** and test on hardware or QEMU, observing serial console output where the banner should appear.

---

## 5. Tips & considerations
- **Placement matters:** choose a point in the board init sequence where the console is available but before too much critical initialization that might change the bootlog layout. `announce_dram_init()` is a good example slot because it runs early but after basic console init.  
- **Binary size:** the banner generator should produce compact code; prefer simple character arrays and a `puts`-style print to keep size impact minimal.  
- **Encoding & fonts:** keep the banner ASCII (or UTF-8-safe) so terminals and serial consoles render predictably.  
- **Kconfig escaping:** if you allow spaces or quotes in `BANNER_TEXT`, add a pre-build script to sanitize/escape the text and generate a safe C string literal.  
- **Integration strategy:** for multiple boards, either embed per-board banner source in each board tree or centralize a generator step that uses `CONFIG_BANNER_TEXT` to produce board-specific outputs during the build.

---

## 6. Example Makefile / build step idea
- A more robust pattern is to add a small pre-build rule that runs the converter and outputs `banner.c`/`banner.h` into the build directory. Example pseudo-Make rule:

```makefile
# pseudo-rule: generate banner sources from CONFIG_BANNER_TEXT
banner_gen:
	./tools/bannergen --text "$(CONFIG_BANNER_TEXT)" -o generated/banner.c
	# ensure generated/banner.c is compiled into the build

# then ensure banner_gen runs before compiling board_f.o
board_f.o: banner_gen
```

This avoids fragile macro quoting and allows arbitrary banner content.

---

## 7. Final notes
- The presented flow is lightweight and easy to integrate into existing U-Boot builds.  
- If you plan to upstream this change or share it across multiple boards, prefer a generator + Kconfig approach so maintainers can set the banner via configuration without editing source files.

---

<!-- Image placeholder: replace the path below with your final banner image or diagram path. This image must be the last item in the post. -->
![banner integration diagram](/path/to/your/image.png)
