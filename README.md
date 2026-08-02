# gbs-OverlayMoveExPlugin

**Version 4.3.0 — Requires GB Studio ≥ 4.3.0**

A GB Studio engine plugin that adds an extended version of the built-in **Overlay Move To** event. The standard event moves the overlay (the Game Boy hardware Window layer) in whole-tile steps at a fixed speed. This plugin adds a single extended event that accepts **pixel-precise target positions** and a **custom movement speed in subpixels per frame**, allowing smooth, fine-grained overlay animations at any speed.

<img width="450" height="94" alt="image" src="https://github.com/user-attachments/assets/7e86d39d-b9fe-4557-b843-93f210880dfd" />

---

## Table of Contents

1. [Concepts](#concepts)
2. [Project Setup](#project-setup)
3. [How to Use](#how-to-use)
4. [Technicalities and Restrictions](#technicalities-and-restrictions)
5. [Events Reference](#events-reference)
6. [Inner Workings](#inner-workings)
7. [Memory Footprint](#memory-footprint)

---

## Concepts

### The GB Studio Overlay (Window Layer)

The Game Boy hardware **Window layer** is a second background plane that slides over the main background. GB Studio uses it as the overlay — primarily the dialogue box frame and the white background behind text. It is positioned by the `win_pos_x` / `win_pos_y` hardware registers (whole pixels). The standard **Overlay Move To** event moves the overlay by a set of hardcoded speed.

### Pixel and Subpixel Positioning

This plugin controls the overlay position using **subpixels** for smooth interpolated movement, while the hardware position is always updated as whole pixels. GB Studio uses a fixed subpixel scale of **32 subpixels per pixel**:

$$\text{subpixels} = \text{pixels} \times 32$$

The **Subpixels per frame** (sppf) speed field therefore means:

| sppf value | Movement speed |
|---|---|
| 32 | 1 pixel per frame |
| 64 | 2 pixels per frame |
| 16 | 0.5 pixels per frame (alternates 0 and 1 px) |
| 1 | 1/32 of a pixel per frame |

The target position is expressed in whole tiles or whole pixels; the plugin converts it to subpixels internally on the first frame of movement.

### Y Before X

The overlay always finishes moving on the Y axis before it begins moving on the X axis. This matches the behaviour of the built-in overlay move event and is a deliberate ordering choice to avoid diagonal movement glitches with the Window hardware register.

---

## Project Setup

1. Copy the plugin folder into your GB Studio project's `plugins/` directory.
2. No additional configuration, engine fields, or compatibility variants are required.

---

## How to Use

1. Add an **Overlay Move To (Extended)** event to any script where you want to animate the overlay position.
2. Set the **X** and **Y** target position. Use the units toggle to choose **tiles** (multiples of 8 pixels) or **pixels**. These can be runtime value expressions or constants.
3. Set **Subpixels per frame** — the movement speed. A value of `32` moves at 1 pixel per frame. Both constants and runtime variable expressions are accepted.
4. The event is **waitable** — the script pauses here each frame until the overlay reaches the target position, then continues.

---

## Technicalities and Restrictions

### Waitable — Blocks Script Execution

The event uses the GBVM `VM_INVOKE` mechanism, which re-evaluates the movement function every frame until the overlay reaches its destination. The script thread that called the event is suspended during movement. Other concurrent script threads continue to run normally.

### Y Axis Finishes Before X Axis Starts

The overlay will always fully reach its target Y position before beginning X-axis movement. If you need simultaneous X+Y movement, it is not directly supported by this event.

### Target Position Is in Pixels or Tiles (Not Subpixels)

The **X** and **Y** fields accept tile or pixel values. They are converted to subpixels internally on the first frame. The **Subpixels per frame** field is the only field in subpixel units.

### Position Is Clamped to Screen Bounds

If a movement step would overshoot either the target position or the screen boundary (`SCREEN_WIDTH_SUBPX` / `SCREEN_HEIGHT_SUBPX`), the position is clamped to the target instead of overshooting. The overlay will never be moved outside the screen area.

### No Engine Files Modified

This plugin only adds a new engine source file (`overlay_move_ex.c`). No existing GB Studio engine files are patched.

---

## Events Reference

### Overlay Move To (Extended)

**Event ID:** `EVENT_OVERLAY_MOVE_TO_FAST`  
**Group:** Screen → Overlay

Moves the overlay to a target position at a custom subpixel speed. The script waits here until movement completes.

| Field | Type | Units | Default | Description |
|---|---|---|---|---|
| X | Value expression | tiles or pixels | 0 | Target horizontal position of the overlay. Toggle the units field between **tiles** (×8 pixels) and **pixels**. |
| Y | Value expression | tiles or pixels | 0 | Target vertical position of the overlay. Same unit toggle as X. |
| Subpixels per frame | Value expression | subpixels/frame | 0 | Movement speed. 32 = 1 pixel per frame. 0 causes no movement. |

**Notes:**
- Y movement completes before X movement begins.
- Setting **Subpixels per frame** to 0 will cause the event to run indefinitely (never reaches the target). Always use a value greater than 0.
- Position values are clamped to screen bounds.

---

## Inner Workings

### The `VM_INVOKE` Mechanism

Unlike `_callNative` (which calls a function once), this event uses `_invoke`, which compiles to a GBVM `VM_INVOKE` instruction. Each frame, after the VM advances the script thread, `vm_overlay_move_to_ex` is called with a reference to a 5-word stack frame that persists between frames. The function returns `TRUE` when movement is complete — at which point `VM_INVOKE` allows the script to advance past the event.

### Stack Frame Layout

The 5-word persistent stack frame used by `vm_overlay_move_to_ex`:

| Slot | Content |
|---|---|
| `stack_frame[0]` | Current X position in subpixels (updated each frame) |
| `stack_frame[1]` | Current Y position in subpixels (updated each frame) |
| `stack_frame[2]` | Target X position in subpixels (set on first call) |
| `stack_frame[3]` | Target Y position in subpixels (set on first call) |
| `stack_frame[4]` | Speed in subpixels per frame (constant, set by the script) |

The JS compile function initialises slots 0 and 1 to `0` (placeholder; overwritten on first call), pushes the pixel-valued X and Y targets into slots 2 and 3, and pushes `sppf` into slot 4:

```js
_stackPushConst(0);                    // slot 0: current x (placeholder)
_stackPushConst(0);                    // slot 1: current y (placeholder)
_stackPushScriptValue(valueX);         // slot 2: target x in pixels
_stackPushScriptValue(valueY);         // slot 3: target y in pixels
_stackPushScriptValue(input.sppf);     // slot 4: speed in subpixels per frame
_invoke("vm_overlay_move_to_ex", 5, ".ARG4");
```

### Tile-to-Pixel Conversion in JS

When the unit toggle is set to **tiles**, the JS compile function multiplies the value by 8 using a left-shift-by-3 expression before pushing it onto the stack:

```js
const scriptValueToPixels = (value, units) => {
  if (units === "pixels") return value;
  return { type: "shl", valueA: value, valueB: { type: "number", value: 3 } };
};
```

This means the runtime value expression is evaluated first and then shifted — the tile-to-pixel conversion is performed at runtime via a GBVM RPN operation, not at compile time.

### First-Frame Initialisation (`start == TRUE`)

On the first call, the `start` parameter is `TRUE`. The function reads the current hardware overlay position from `win_pos_x` / `win_pos_y` (in pixels) and converts them to subpixels, storing them in `stack_frame[0]` and `stack_frame[1]`. It also converts the pixel-valued target positions in slots 2 and 3 to subpixels:

```c
if (start) {
    stack_frame[0] = PX_TO_SUBPX(win_pos_x);    // convert current pos to subpx
    stack_frame[1] = PX_TO_SUBPX(win_pos_y);
    stack_frame[2] = PX_TO_SUBPX(stack_frame[2]);// convert target x to subpx
    stack_frame[3] = PX_TO_SUBPX(stack_frame[3]);// convert target y to subpx
}
```

### Per-Frame Movement Logic

Each frame, Y is evaluated first. If the current Y subpixel position differs from the target:

```c
if (win_pos_subpx < target_win_pos_subpx) {
    win_pos_subpx += stack_frame[4];  // add speed
    if (win_pos_subpx > target_win_pos_subpx || win_pos_subpx > SCREEN_HEIGHT_SUBPX)
        win_pos_subpx = target_win_pos_subpx;  // clamp
} else {
    win_pos_subpx -= stack_frame[4];  // subtract speed
    if (win_pos_subpx < target_win_pos_subpx || win_pos_subpx > SCREEN_HEIGHT_SUBPX)
        win_pos_subpx = target_win_pos_subpx;  // clamp (underflow check via >)
}
((SCRIPT_CTX *)THIS)->waitable = TRUE;
flag = FALSE;
win_pos_y = win_dest_pos_y = SUBPX_TO_PX(win_pos_subpx);
stack_frame[1] = win_pos_subpx;
```

The updated subpixel value is stored back in `stack_frame[1]`, and the hardware pixel position is updated immediately via `SUBPX_TO_PX` (integer division by 32). The same logic runs for X after Y.

If both axes are already at their targets on entry, `flag` remains `TRUE` and the function returns `TRUE`, allowing the script to advance.

Note: the underflow check `win_pos_subpx > SCREEN_HEIGHT_SUBPX` after subtraction detects unsigned integer wrap-around (when subpixel position wraps from 0 back to a large value), clamping it to the target safely.


---

## Memory Footprint

Measured against the stock GB Studio **4.3.0-e1** engine (per-file SDCC compile with GB Studio's build flags, default engine settings). Values are the plugin's *delta* versus the stock engine; DMG build, with CGB noted where it differs. ROM cost lands in banked ROM (GB Studio's autobanker spreads it across switchable banks); using the plugin's events additionally compiles a few bytes of GBVM script per call into your project's script banks.

| | Cost |
|---|---|
| WRAM | +0 bytes |
| ROM | +663 bytes |

- **WRAM:** no change — the plugin drives the engine's existing overlay state.
- **Engine WRAM headroom:** the stock GB Studio 4.3.0 engine leaves about **854 bytes** of WRAM free (usable engine WRAM is 7,776 bytes at 0xC0A0–0xDF00; the stock engine uses 6,922 bytes). With this plugin installed roughly **854 bytes** remain. This figure does not depend on how many global variables your project defines: the script memory array has a fixed size of VM_HEAP_SIZE + (VM_MAX_CONTEXTS × VM_CONTEXT_STACK_SIZE) words — 768 + 16 × 64 = 1,792 words (3,584 bytes) with stock engine settings.
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
| Bank 0 free with this plugin installed | **1,451** of 16,384 (91% used) |

**This plugin costs nothing in bank 0.** All of its code lives in a switchable
ROM bank; nothing it adds is resident in bank 0.

<details><summary>How this was measured</summary>

GB Studio 4.3.2, DMG target, default engine settings. Each module's bank 0
contribution is the `A _HOME size` record that SDCC writes into its `.rel`
object, summed over the engine sources this plugin provides. Stock sizes come
from building projects whose only plugin ships no engine C, so every module in
them is the untouched engine; two such builds were compared and agreed on all
73 shared modules.

The "free" figure is a stock project with this plugin and nothing else. Your
own number will differ: other plugins, and any engine settings that change what
the core compiles, move it independently of this plugin.

</details>
<!-- BANK0:END -->
