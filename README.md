# gbs-StreamableActorPlugin

**Version 4.3.0 — Requires GB Studio ≥ 4.3.0**

An engine plugin that streams an actor's animation frames straight into sprite VRAM, one frame at a time. A spritesheet then costs the VRAM of its **largest single frame** instead of the VRAM of **all its frames**, no matter how many frames, directions or animation states it has.

Frames may have completely different layouts and different tile counts; the streamer copies whatever the current frame needs.


https://github.com/user-attachments/assets/a34a7e8c-7b4e-423a-ac06-ebe73a1ccc8f


---

## Table of Contents

1. [Concepts](#concepts)
2. [Project Setup](#project-setup)
3. [Engine Settings](#engine-settings)
4. [Size Limits and Restrictions](#size-limits-and-restrictions)
5. [Events Reference](#events-reference)
6. [Engine Fields Reference](#engine-fields-reference)
7. [Media](#media)
8. [Memory Footprint](#memory-footprint)
9. [Bank 0 (HOME) Usage](#bank-0-home-usage)
10. [Changelog](#changelog)

---

## Concepts

### How GB Studio normally spends sprite VRAM

The Game Boy has room for 256 sprite tiles, 128 of it shared with background tiles and 192 if using GBStudio's ui tiles, double in colour-only mode. At scene load GB Studio uploads **every tile of every spritesheet used in the scene**. A 24-frame player sheet with 40 unique tiles occupies 40 tile slots for the whole scene, even though only 4 to 8 of those tiles are on screen at any moment.

### What this plugin does

- **When the project is built**, it re-packs the chosen spritesheet so every frame owns a contiguous block of tiles. Identical blocks are shared, so repeated frames cost nothing extra.
- **At scene load**, the actor gets an exclusive VRAM band exactly as big as the sheet's largest frame, and the streamed sheet is kept **out** of the scene's shared sprite pool — no tiles are uploaded for it.
- **At run time**, the streamer notices the actor's frame changing and copies that frame's block into the band. Nothing about how the actor is drawn changes, so all the stock animation events keep working.

Uploads happen only in a VBlank that a render pass has just fed, so the new sprite layout and the new tile data become visible on the same displayed frame — no tearing between which tiles are drawn and what is in them.

---

## Project Setup

1. Copy `src/StreamableActorPlugin` into your project's `plugins/` folder. No stock engine file is modified, so there are no compatibility variants and no conflicts with other engine plugins.
2. **Give the actor a small placeholder sprite in the editor.** The actor's editor spritesheet is still loaded at scene load, so use a tiny one — a 1-frame 16×16 sprite is 4 tiles. The placeholder is only visible until the first frame is streamed.
3. Add **Stream Actor Spritesheet** to the scene's *On Init*, or the actor's *On Init*, and pick the actor and the real spritesheet.
4. Build. That is all — the actor now animates normally with every stock event (*Set Animation State*, *Set Animation Frame*, movement, direction changes), and the streamer follows whatever changes the frame.

For an actor whose index is only known at run time, use **Reserve Streamed Actor Tiles** (build time, picks the actor) together with **Stream Actor Spritesheet By Index** (run time).

---

## Engine Settings

Found under **Settings → Engine → Streamable Actors**.

| Setting | Default | Description |
|---|---|---|
| **Streaming enabled** | on | Master switch for the streamer. |
| **Tiles uploaded per frame** | 8 | The per-VBlank upload budget, in 8×8 tiles. |
| **Streamed actor slots** | 4 | How many actors can stream at once. |

---

## Size Limits and Restrictions

### The VBlank budget

Copying tiles takes time, and VBlank is short. The default budget of 8 tiles leaves comfortable headroom; a Game Boy Color in double-speed mode can take about twice that. If several actors change frame at once and the budget runs out, the actors that did not fit keep showing their previous frame and are served **first** on the next frame — animation lags by a frame, nothing is corrupted. Raising the budget too far overruns VBlank and corrupts tiles.

A frame bigger than the whole budget is uploaded anyway, blocking the other slots for that VBlank, so an under-sized budget never freezes an animation permanently.

### The actor's editor spritesheet still costs VRAM

The actor's editor sheet is uploaded into the reserved band before any script runs. If that sheet is bigger than the streaming band, the band grows to match it — which defeats the purpose. Always give streamed actors a stub sprite.

### ROM size

Per-frame blocks cannot share tiles *between* frames, so a sheet whose frames overlap heavily — a large sprite where only a hand moves — grows in ROM. Byte-identical frames still share a block, and sheets with little cross-frame duplication cost the same as before. In the example project both sprites generate exactly the same number of tile bytes as their stock tilesets while dropping from 24 to 4 VRAM tiles.

If the streamed sheet is not used by anything else in the project, GB Studio still emits its normal tileset as well; that copy is unused by streamed actors.

### 8×16 sprite mode

In 8×16 mode tiles are allocated in pairs, so a band is always an even number of tiles. Nothing to configure.

### Colour sprites

Colour-only sheets normally split their tiles between both VRAM banks. Streamed frames always land in VRAM bank 0, since a streaming band is small enough not to need the second bank.

### Scene changes

Streaming slots are validated every VBlank, and a slot is only serviced while the actor still points at the streamed sheet and still owns the same tile band. Registrations left behind by a previous scene are therefore ignored and their slots recycled, rather than writing into another actor's tiles. **Stop All Actor Streaming** is available if you want the slots back immediately.

### Save states

Streaming registrations are not saved. Re-run the *Stream Actor Spritesheet* events on load — they are normally in *On Init*, which runs anyway.

### Stock spritesheet events on a streamed actor

A stock *Set Actor Spritesheet* pointed at a streamed sheet would upload nothing. Use *Stream Actor Spritesheet* again to switch a streamed actor to another streamed sheet. Using the stock event with a **normal** sheet stops the streaming automatically, but the band is only as big as it was reserved.

---

## Events Reference

All events live under **Actor → Streaming**.

---

### Stream Actor Spritesheet

**`EVENT_STREAM_ACTOR_SPRITESHEET`**

Re-packs the sheet when the project is built, reserves the actor's band, and binds the actor to the streamed sheet at run time.

| Field | Default | Description |
|---|---|---|
| Actor | Self | Actor that will stream its frames. |
| Sprite Sheet | Last sprite | Sheet to stream. |
| Animation State | Default | State to select on the streamed sheet. |
| Reserve tiles (0 = auto) | 0 | Band size; 0 uses the sheet's largest frame. |
| Upload first frame immediately | On | Waits for VBlank and uploads the current frame at once. |
| Apply sheet collision bounds | On | Copies the sheet's bounds onto the actor. |

---

### Stream Actor Spritesheet By Index

**`EVENT_STREAM_ACTOR_SPRITESHEET_BY_INDEX`**

The same, for an actor selected by run-time index (0 = player). Does **not** reserve VRAM — pair it with *Reserve Streamed Actor Tiles*.

| Field | Default | Description |
|---|---|---|
| Actor index | 0 | Scene actor index; 0 = player. |
| Sprite Sheet | Last sprite | Sheet to stream. |
| Animation State | Default | State to select. |
| Band size limit (0 = auto) | 0 | Never upload more than this many tiles. |
| Upload first frame immediately | On | Waits for VBlank and uploads the current frame at once. |
| Apply sheet collision bounds | On | Copies the sheet's bounds onto the actor. |

---

### Reserve Streamed Actor Tiles

**`EVENT_STREAM_ACTOR_RESERVE_TILES`**

Build-time only; emits no runtime code. Gives an actor an exclusive sprite VRAM band and removes the streamed sheet — and the actor's own stub sheet, when nothing else needs it — from the scene's shared sprite pool.

| Field | Default | Description |
|---|---|---|
| Actor | Self | Actor to reserve a band for. |
| Sprite Sheet | Last sprite | Sheet used to size the band. |
| Reserve tiles (0 = from sheet) | 0 | Explicit band size. Use the largest frame of the biggest sheet the actor will ever stream. |

---

### Stop Streaming Actor

**`EVENT_STREAM_ACTOR_STOP`**

Releases the actor's streaming slot. VRAM keeps the last streamed frame.

---

### Stop All Actor Streaming

**`EVENT_STREAM_ACTOR_STOP_ALL`**

Clears every streaming slot.

---

### Upload Streamed Actor Frame

**`EVENT_STREAM_ACTOR_UPLOAD_NOW`**

Waits for VBlank and uploads the actor's current frame immediately, instead of waiting for the streamer. Costs one frame.

---

### Streamed Actor Set Enabled

**`EVENT_STREAM_ACTOR_SET_ENABLED`**

Suspends or resumes the whole streamer. Useful around code that needs the entire VBlank.

---

### Streamed Actor Set Tile Budget

**`EVENT_STREAM_ACTOR_SET_BUDGET`**

Changes the per-VBlank tile budget at run time.

---

### Streamed Actor Get Info

**`EVENT_STREAM_ACTOR_GET_INFO`**

Writes *is streaming* (0/1), *band base tile* and *band size* into three variables.

---

## Engine Fields Reference

These runtime fields back the two settings events and can also be read or written directly via **Engine Field Value**.

| Field | Description |
|---|---|
| `streamable_actor_enabled` | Whether the streamer is currently running. |
| `streamable_actor_budget` | The per-VBlank upload budget, in tiles. |

---

## Media

`StreamableActorExample/` is a working project: the player, Link and an NPC are placed with a 4-tile stub sprite and stream their sheets into 4-tile bands, so the scene's shared sprite pool ends up empty and the three actors together use 12 sprite tiles.

The example's Link actor is driven by his real tile sheet and animation table, imported from the [LADX disassembly](https://github.com/zladx/LADX-Disassembly). Press **A / B** in the demo to step through all 41 imported actions — walk, push, shield, lift, pull, swim, jump, falling and more — with a dialogue naming each action as it changes.

That sheet is 102 unique 8×16 tiles, more than the Game Boy can hold at once; the stock loader simply cannot fit it. Streamed, Link costs **4 tiles**, and frames whose two halves share one tile pair upload only 2.

> The imported graphics are Nintendo's and are not checked in. `tools/import_ladx_link.js` and `tools/gen_link_demo_scene.js` rebuild them from your own disassembly checkout.

---

<!-- SETTINGCOST:BEGIN -->
### What each engine setting costs

Every setting here changes what gets compiled. Figures are what you **get back by
turning the setting off**; rows marked *off by default* show what turning it **on**
costs instead, and sliders show the cost per step. A dash means that budget does not
move.

| Setting | Bank 0 | WRAM | Banked ROM |
|---|---|---|---|
| Streamed actor slots *(slider 1–16, default 4)* | — | 13 B/step | — |

- **Streamed actor slots**: going from 1 to 16 moves WRAM by +195 B.

<details><summary>How these were measured</summary>

GB Studio 4.3.0-e1. This plugin's `engine/src/**/*.c` was compiled with the
toolchain and flags GB Studio itself uses (`lcc -msm83:gb -Wf--max-allocs-per-node 3000
-DHUGE_TRACKER -DRUMBLE_ENABLE=0x08u`) against a merged include tree, and the SDCC object
files' area records were read: `_HOME` is bank 0, `_DATA`/`_INITIALIZED`/`_BSS` are WRAM,
and `_CODE*`/`_CONST`/`_LIT`/`_INITIALIZER` are banked ROM.

Two caveats. Only this plugin's own engine sources are measured, so a setting that also
changes a struct shared with stock engine files can move a few more bytes in files the
plugin does not ship. And each setting is toggled on its own: a handful measure slightly
*negative* because enabling their code lets the compiler drop a fallback path elsewhere,
and settings that gate other settings only show their own contribution.

</details>
<!-- SETTINGCOST:END -->

## Memory Footprint

Measured from the example project with the default 4 streaming slots.

| | Cost |
|---|---|
| WRAM | +56 bytes |
| ROM (bank 0) | +89 bytes |
| ROM (banked) | the streamer, the upload helper and the VM helpers, plus one re-packed tile block per distinct frame of each streamed sheet |
| SRAM | none |

- **WRAM:** 56 bytes — 2 engine fields, 4 slots of 13 bytes, and 2 bytes of state. Fewer slots cost proportionally less.
- **Bank 0:** only 89 bytes are resident; the streamer itself runs from the plugin's own switchable bank. See [Bank 0 (HOME) Usage](#bank-0-home-usage).
- **Sprite VRAM saved** per streamed actor is the whole sheet's tile count minus its largest frame's tile count — in the example, 24 tiles down to 4 for a 12-frame 16×16 sheet.

---

<!-- BANK0:BEGIN -->
## Bank 0 (HOME) Usage

Bank 0 is the 16 KB non-switchable ROM bank that the GB Studio engine core,
the interrupt handlers and the GBDK runtime all share. Banked ROM is cheap
(add another bank), bank 0 is not, so it is usually the first thing a project
runs out of.

| | Bytes |
|---|---|
| Bank 0 used by this plugin | **+89** |
| Bank 0 free with this plugin installed | **1,362** of 16,384 (92% used) |

Everything else this plugin adds lives in banked ROM.

| Module | This plugin | Stock engine | Bank 0 cost |
|---|---|---|---|
| `streamable_actor.c` | 89 | — | +89 |

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

## Changelog

Grouped by the date each change was merged into the official
[gb-studio-plugins](https://github.com/gb-studio-dev/gb-studio-plugins) repository.

Only bug fixes, new features and feature changes are listed. Engine version
bumps, patch regeneration, packaging fixes and documentation edits are omitted.

### 2026-08-02

- Initial release: LADX-style per-frame sprite streaming into a small reserved VRAM band.
