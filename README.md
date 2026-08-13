# gbs-OverlayMoveExPlugin

**Version 4.3.0 — Requires GB Studio ≥ 4.3.0**

A GB Studio engine plugin that adds an extended version of the built-in **Overlay Move To** event. The standard event moves the overlay in whole-tile steps at a fixed speed. This plugin adds a single extended event that accepts **pixel-precise target positions** and a **custom movement speed in subpixels per frame**, allowing smooth, fine-grained overlay animations at any speed.

<img width="450" height="94" alt="image" src="https://github.com/user-attachments/assets/7e86d39d-b9fe-4557-b843-93f210880dfd" />

---

## Table of Contents

1. [Concepts](#concepts)
2. [Project Setup](#project-setup)
3. [Size Limits and Restrictions](#size-limits-and-restrictions)
4. [Events Reference](#events-reference)
5. [Memory Footprint](#memory-footprint)
6. [Bank 0 (HOME) Usage](#bank-0-home-usage)
7. [Changelog](#changelog)

---

## Concepts

### The overlay

The overlay is the Game Boy's Window layer — a second background plane that slides over the main background. GB Studio uses it for the dialogue box frame and the white background behind text. The standard **Overlay Move To** event moves it at a fixed, hardcoded speed.

### Pixel and subpixel positioning

This plugin moves the overlay in **subpixels** for smooth interpolated movement, while the hardware position is always a whole pixel. GB Studio uses 32 subpixels per pixel, so the **Subpixels per frame** speed field means:

| Value | Movement speed |
|---|---|
| 32 | 1 pixel per frame |
| 64 | 2 pixels per frame |
| 16 | 0.5 pixels per frame (alternates 0 and 1 px) |
| 1 | 1/32 of a pixel per frame |

The target position is given in whole tiles or whole pixels; only the speed field is in subpixels.

### Y before X

The overlay always finishes moving on the Y axis before it starts moving on the X axis. This matches the built-in overlay move event, and is deliberate — it avoids diagonal movement glitches with the Window hardware.

---

## Project Setup

1. Copy the plugin folder into your GB Studio project's `plugins/` directory. No additional configuration, engine fields, or compatibility variants are required.
2. Add an **Overlay Move To (Extended)** event to any script where you want to animate the overlay position.
3. Set the **X** and **Y** target position, using the units toggle to choose **tiles** or **pixels**. Both can be constants or runtime expressions.
4. Set **Subpixels per frame** — the movement speed; 32 moves at 1 pixel per frame.

The event is waitable: the script pauses until the overlay reaches the target position, then continues.

---

## Size Limits and Restrictions

### The event blocks its script

The calling script thread is suspended until the overlay reaches its destination. Other concurrent script threads continue to run normally.

### Y finishes before X starts

The overlay always fully reaches its target Y position before beginning X-axis movement. Simultaneous X and Y movement is not supported by this event.

### Never set the speed to 0

With **Subpixels per frame** at 0 the overlay never reaches its target, so the event never completes and the script hangs. Always use a value greater than 0.

### Position is clamped to the screen

A movement step that would overshoot the target, or leave the screen area, is clamped to the target instead. The overlay is never moved outside the screen.

### No engine files modified

The plugin only adds a new engine source file, so it has no compatibility conflicts with other engine plugins.

---

## Events Reference

---

### Overlay Move To (Extended)

**`EVENT_OVERLAY_MOVE_TO_FAST`** — group: **Screen → Overlay**

Moves the overlay to a target position at a custom subpixel speed. The script waits here until movement completes.

| Field | Default | Description |
|---|---|---|
| X | 0 | Target horizontal position of the overlay. Toggle the units between **tiles** and **pixels**. Accepts a value, variable or expression. |
| Y | 0 | Target vertical position of the overlay, with the same unit toggle. |
| Subpixels per frame | 0 | Movement speed; 32 = 1 pixel per frame. Must be greater than 0 or the event never finishes. |

---

## Memory Footprint

Measured against the stock GB Studio **4.3.0-e1** engine by `measure_plugin_memory.js` (per-file SDCC compile with GB Studio's own build flags, at default engine settings; report of 2026-08-13). Figures are this plugin's *delta* versus stock — a file that replaces a stock engine file counts only the difference, which is why a plugin can come out negative. Using the plugin's events additionally compiles a few bytes of GBVM script per call into your project's script banks, on top of the fixed cost below.

| Budget | Cost |
|---|---|
| Bank 0 (HOME) | 0 bytes |
| WRAM | 0 bytes |
| Banked ROM | +663 bytes |

- **Bank 0:** nothing. Every function the plugin adds is compiled into a switchable ROM bank.
- **WRAM:** no change — the plugin drives the engine's existing overlay state.
- **Banked ROM:** 663 bytes for the extended overlay movement helpers.
- **Engine WRAM headroom:** a stock GB Studio 4.3.0 project leaves about **854 bytes** of WRAM free (usable engine WRAM is 7,776 bytes at 0xC0A0–0xDF00; the stock engine uses 6,922). With this plugin installed roughly **854 bytes** remain. That does not change with the number of global variables your project defines: the script memory array is a fixed 3,584 bytes at stock engine settings (VM_HEAP_SIZE + VM_MAX_CONTEXTS × VM_CONTEXT_STACK_SIZE = 768 + 16 × 64 words).
- **SRAM:** not used.

---

<!-- BANK0:BEGIN -->
## Bank 0 (HOME) Usage

Bank 0 is the 16 KB non-switchable ROM bank that the GB Studio engine core,
the interrupt handlers and the GBDK runtime all share. Banked ROM is cheap
(add another bank), bank 0 is not, so it is usually the first thing a project
runs out of.

| | Bytes |
|---|---|
| Bank 0 used by this plugin | **0** |

**This plugin costs nothing in bank 0.** Every one of its functions is compiled
into a switchable ROM bank; nothing it adds is resident in bank 0.
<!-- BANK0:END -->

## Changelog

Grouped by the date each change was merged into the official
[gb-studio-plugins](https://github.com/gb-studio-dev/gb-studio-plugins) repository.

Only bug fixes, new features and feature changes are listed. Engine version
bumps, patch regeneration, packaging fixes and documentation edits are omitted.

### 2026-02-09

- Initial release.
