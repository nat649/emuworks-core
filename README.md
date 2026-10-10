# EmuWorks Core

Original MIT-licensed calculator firmware for [EmuWorks](https://github.com/nat649/emuworks). Core runs real ARM machine code on an emulated Cortex-M7, using the LCD bus and GPIO keyboard matrix.

This repository contains the firmware sources and compiled images. The Windows emulator is maintained separately and embeds a version of Core, so users do not need to download this repository to use it.

## Features

- Basic arithmetic: `+`, `-`, `*`, `/`, decimals, parentheses, unary signs, and operator precedence.
- Integer powers from -100 to 100. Powers associate to the right: `2^3^2 = 512`; `-2^2 = -4`.
- `ANS` to reuse the last result.
- Eight-entry calculation history for the current session.
- A 320 x 240 interface with rounded cards and original glyphs, rasterized at build time.
- Partial screen updates: rendering is composed in SRAM before completed pixels are sent to the LCD, avoiding full-screen clearing during input.

Version 0.1 uses single-precision floating-point arithmetic and displays six significant digits. Input decimal exponents are limited to -30 through 30, and result magnitudes to approximately 1e30. Very small values may round to zero. Python, graphing, trigonometric functions, symbolic algebra, and persistent history are not implemented.

## Use Core in EmuWorks

Select **emuworks-core-0.1** in EmuWorks, install it, and start the emulator. Core is installed automatically on a fresh installation without firmware. Existing user firmware is preserved.

| PC key | Action |
|---|---|
| Enter | Evaluate the expression |
| Escape | Clear the expression and displayed result |
| Backspace | Delete the character before the cursor |
| Left / Right | Move the cursor |
| Up / Down | Recall calculation history |
| A | Insert `ANS` |

History is cleared when the firmware restarts.

**Emulator target only:** do not flash these images onto a physical calculator. Startup assumes that the memory and display bus are already exposed by the Renode platform; it does not initialize all physical hardware.

## Build

Requirements:

- Node.js.
- Arm GNU Toolchain with `arm-none-eabi-gcc`, `objcopy`, `size`, and `nm`.
- Git to clone the repositories and record firmware updates.

From this repository's root:

```powershell
# Optional: set this if the toolchain is not on PATH.
$env:ARM_GCC_BIN = 'C:\path\to\arm-toolchain\bin'
node build.cjs
```

The build produces:

| File | Purpose |
|---|---|
| `build/core.elf` | Firmware with symbols |
| `build/internal.bin` | Firmware image for internal flash |
| `build/external.bin` | Identification data for the emulator's two-image import format |
| `build/tests.elf`, `build/tests.bin` | Separate ARM test program |

All executable firmware code fits in internal flash. The external image does not contain calculator code.

The build was validated with GCC Arm 12.2 MPACBTI-Rel1. It targets Cortex-M7, Thumb, and the single-precision FPU with `-ffreestanding -nostdlib -fno-builtin`. No C runtime, libgcc, or math library is linked.

To update the tracked images in `bundled/`:

```powershell
node build.cjs --bundle
```

These images let the emulator application compile without an ARM toolchain. Updating them here does not automatically update the emulator repository.

## Test with Renode

Install Renode 1.16 and clone the [EmuWorks repository](https://github.com/nat649/emuworks) alongside this one. Build Core first, then run this command from the Core repository:

```powershell
node ../emuworks/tests/core.integration.cjs 'C:\Program Files\Renode\bin\Renode.exe' .
```

The integration runner uses EmuWorks' peripheral models and this repository's build output. It checks:

- 45 arithmetic, error-handling, and formatting cases on the emulated processor.
- Keyboard input, operator precedence, history recall, division by zero, and powers.
- Rapid digit entry, unchanged static screen regions, and LCD pixel traffic.

It also writes a screenshot to the temporary test directory. The test firmware is separate from the image embedded in the application. Renode's test directory must have a path without spaces.

## Update the firmware embedded in EmuWorks

1. Run `node build.cjs --bundle` in this repository and run the integration tests.
2. Commit the source changes and both images in `bundled/`. The Core working tree must be clean before importing.
3. From the emulator repository, import the committed images:

   ```powershell
   node tools/update-core.cjs ../emuworks-core
   ```

4. Rebuild the Windows application with the .NET 8 SDK:

   ```powershell
   dotnet publish app/EmuWorks.csproj -c Release -r win-x64 --self-contained false -p:PublishSingleFile=true
   ```

5. Commit the imported firmware and provenance changes in the emulator repository.

The import script copies the committed images and MIT license. It records the source commit and SHA-256 hashes in `firmware/core/source.json`, so each emulator build can identify the exact Core revision it includes. No Git submodule or runtime download is required.

## Source layout

| File | Responsibility |
|---|---|
| `startup.S`, `linker.ld` | ARM startup and memory layout |
| `main.c` | Keyboard scanning, UI, LCD output, and session history |
| `calc.c`, `calc.h` | Expression evaluation and number formatting |
| `font.cjs` | Original glyph definitions and build-time rasterization |
| `build.cjs` | Firmware and test image compilation |
| `tests.c` | On-device calculation tests |
| `bundled/` | Compiled images available for integration |

## License

[MIT](LICENSE), copyright 2026 nat649.

Core's startup code, calculator, drivers, and glyphs are original project code. No Epsilon/Omega code, fonts, images, or binaries are included. No NumWorks logo is embedded. EmuWorks Core is an independent project; third-party tools and firmware retain their own licenses.
