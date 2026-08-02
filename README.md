# gbs-StreamableActorPlugin

**Requires GB Studio ≥ 4.3.0**

An engine plugin that streams an actor's animation frames straight into sprite VRAM, one frame at a time — the way *The Legend of Zelda: Link's Awakening* streams Link's frames into a fixed tile band. A spritesheet then costs the VRAM of its **largest single frame** instead of the VRAM of **all its frames**, no matter how many frames, directions or animation states it has.

Frames may have completely different metasprite/OAM layouts and different tile counts; the streamer copies whatever the current frame needs.

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

### How GB Studio normally spends sprite VRAM

The Game Boy has room for 256 sprite tiles (192 usable by GB Studio in practice, 128 in colour-only mode). At scene load GB Studio uploads **every tile of every spritesheet used in the scene** and gives each actor a `base_tile` pointing at its sheet. A 24-frame player sheet with 40 unique tiles occupies 40 tile slots for the whole scene, even though only 4 to 8 of those tiles are on screen at any moment.

### How Link's Awakening does it

LADX keeps a small fixed band of sprite tiles per entity and DMAs the tiles of the *current* frame into that band every time the frame changes. ROM holds one contiguous tile block per frame; VRAM only ever holds the frame being drawn. That is what lets an 8×16-pixel hero have dozens of animation frames on a machine with 256 tile slots.

### What this plugin does

* **At build time** it re-packs the chosen spritesheet so every frame owns a contiguous block of tiles, and rewrites that frame's metasprite so it references tiles `0..n-1` of its own block. Identical blocks are shared, so repeated frames cost nothing extra.
* **At scene load** the actor gets an exclusive VRAM band exactly as big as the sheet's largest frame (the plugin sets the same `actorsExclusiveLookup` reservation the stock *Set Actor Spritesheet* scan uses), and the streamed sheet itself is kept **out** of the scene's shared sprite pool — no tiles are uploaded for it.
* **At run time** a VBlank handler notices `actor->frame` changing and copies that frame's block into the band. The actor's `base_tile` never changes, so nothing about the stock render path changes.

The handler only ever uploads in a VBlank that a render pass just fed, so the new OAM layout and the new tile data become visible on the same displayed frame — no tearing between "which tiles are drawn" and "what is in them". See [Frame sync](#run-time-staying-in-sync-with-oam) for how that is detected.

---

## Project Setup

1. Copy `src/StreamableActorPlugin` into your project's `plugins/` folder (e.g. `plugins/Mico27/StreamableActorPlugin`).
2. No stock engine file is modified, so there are **no `engineAlt` variants** and no conflicts with other engine plugins.
3. Optional settings live under **Settings → Engine → Streamable Actors**:
   * **Streaming enabled** — master switch (default on).
   * **Tiles uploaded per frame** — VBlank budget in 8×8 tiles (default 8).
   * **Streamed actor slots** — how many actors can stream at once (default 4).

---

## How to Use

1. **Give the actor a small placeholder sprite in the editor.** The actor's editor spritesheet is still loaded at scene load, so use a tiny one (a 1-frame 16×16 sprite = 4 tiles). That placeholder is only visible until the first frame is streamed.
2. Add **Stream Actor Spritesheet** to the scene's *On Init* (or the actor's *On Init*), pick the actor and the real spritesheet.
3. Build. That is all — the actor now animates normally with every stock event (*Set Animation State*, *Set Animation Frame*, movement, direction changes); the plugin follows `actor->frame` whatever changes it.

For an actor whose index is only known at run time, use **Reserve Streamed Actor Tiles** (build time, picks the actor) together with **Stream Actor Spritesheet By Index** (run time).

See `StreamableActorExample/` for a working project: the player, Link and an NPC are placed with a 4-tile stub sprite and stream their sheets into 4-tile bands, so the scene's shared sprite pool ends up empty (`n_sprites = 0`) and the three actors together use 12 sprite tiles.

---

## Example: Link's Awakening animations

The example project's Link actor is driven by the real thing — his tile sheet and animation table imported straight from the [LADX disassembly](https://github.com/zladx/LADX-Disassembly). Press **A / B** in the demo to step through all 41 imported actions (walk, push, shield, lift, pull, swim, jump, falling, …); a dialogue names the action as it changes.

That sheet is **102 unique 8×16 tiles = 204 sprite tiles**, more than the Game Boy can hold at once — the stock GB Studio loader simply cannot fit it. Streamed, Link costs **4 tiles**, and frames whose two halves share one tile pair upload only 2.

### Regenerating the assets

The imported graphics are Nintendo's and are **not** checked in (see `.gitignore`). To rebuild them from your own disassembly checkout:

```bash
node tools/import_ladx_link.js --ladx /path/to/LADX-Disassembly --preview
```

```bash
node tools/gen_link_demo_scene.js
```

`import_ladx_link.js` reads three files from the disassembly and writes `assets/sprites/ladx_link.png(.gbsres)` plus `tools/ladx_link_actions.json`:

| Source | Used for |
|---|---|
| `src/gfx/characters/oam_link_2.dmg.png` | the animation tile sheet |
| `src/code/bank20.asm` | `LinkAnimationStateTable` (two tile indices per state) and the OAM attribute table beside it (X/Y flips) |
| `src/constants/gameplay.asm` | the `LINK_ANIMATION_STATE_*` names, grouped into playable actions |

`gen_link_demo_scene.js` then rewrites the demo scene from that action list: the A/B handler, the chained `EVENT_SWITCH` that selects each action's animation state, and the dialogue for each one.

### Why the mapping is 1:1

LADX draws Link as exactly two 8×16 hardware sprites at `(x, y)` and `(x + 8, y)` using **fixed** OBJ tiles 0 and 2 (`DrawLinkSprite`, `src/code/home/animated_tiles.asm`). The animation state never changes the tile ids — it changes which tiles get copied into VRAM (`func_020_54F5`, `src/code/bank20.asm`). That is precisely what this plugin does, so every LADX animation state becomes one streamed GB Studio frame, and each of its two halves becomes one metasprite tile with the flips from the attribute table.

Three subtleties worth recording:

- The table byte is a **tile** index, not a tile-pair index — the lookup shifts it left four times (× `$10`, one tile) and then copies `$20` bytes.
- Because `gfx.py` builds these sheets with `--interleave`, a tile and the tile directly below it are consecutive in the 2bpp, so tile index `t` is the 8×16 block at column `(t/2) % 8`, row pair `(t/2) / 8` of the PNG.
- The sheets are drawn on a **black** background and LADX's OBJ palette runs the opposite way to GB Studio's: greyscale sample 0 is the transparent background, and *brighter* samples are *darker* shades on screen. The importer therefore flips the three visible shades (`1 → dark, 2 → mid, 3 → light`) and writes an 8-bit indexed PNG using GB Studio's own sprite palette — `#65FF00` transparent, `#E0F8CF` light, `#86C06C` mid, `#071821` dark — the same colours as the stock sprite assets.

---

## Technicalities and Restrictions

### The VBlank budget

Copying tiles takes about 6 CPU cycles per byte, and VBlank is roughly 1140 cycles on DMG. The default budget of 8 tiles (128 bytes) leaves comfortable headroom; the Game Boy Color in double-speed mode can take about twice that. If several actors change frame on the same frame and the budget runs out, the actors that did not fit keep showing their previous frame and are served **first** on the next frame — animation lags by a frame, nothing is corrupted. Raising the budget too far overruns VBlank and corrupts tiles.

A frame that is bigger than the whole budget is uploaded anyway (and blocks the other slots for that VBlank), so an under-sized budget never freezes an animation permanently.

### The actor's editor spritesheet still costs VRAM

`load_scene()` uploads the actor's editor sheet into the reserved band before any script runs. If that sheet is bigger than the streaming band, the band grows to the sheet's size — which defeats the purpose. Always give streamed actors a stub sprite.

### ROM size

Per-frame blocks cannot share tiles *between* frames, so a sheet whose frames overlap heavily (large sprites where only a hand moves) grows in ROM. Frames that are byte-identical still share a block, and sheets with little cross-frame duplication cost the same as before (both sprites in the example project generate exactly the same number of tile bytes as their stock tilesets, 384 each, while dropping from 24 to 4 VRAM tiles).

If the streamed sheet is not used by anything else in the project, GB Studio still emits its normal tileset as well; that copy is unused by streamed actors.

### 8×16 sprite mode

In 8×16 mode tiles are allocated in pairs and frame-local tile ids are always even, so a band is always an even number of tiles. Nothing to configure.

### Colour sprites

Colour-only sheets normally split their tiles between VRAM banks 0 and 1. Streamed frames always land in VRAM bank 0 and the `S_VRAM2` attribute bit is cleared in the re-packed metasprites, since a streaming band is small enough not to need the second bank.

### Scene changes

Streaming slots are validated every VBlank: a slot is only serviced while the actor still points at the streamed sheet **and** still owns the same tile band. Registrations left behind by a previous scene are therefore ignored (and their slots recycled) instead of writing into another actor's tiles. **Stop All Actor Streaming** is available if you want the slots back immediately.

### Save states

Streaming slots hold RAM pointers and are not saved. Re-run the *Stream Actor Spritesheet* events on load (they are normally in *On Init*, which runs anyway).

### Stock spritesheet events on a streamed actor

The generated sheet declares no tileset, so a stock *Set Actor Spritesheet* pointed at it would upload nothing. Use *Stream Actor Spritesheet* again to switch a streamed actor to another streamed sheet; using the stock event with a **normal** sheet stops the streaming automatically (the slot's staleness check fails) but the band is only as big as it was reserved.

---

## Events Reference

All events live under **Actor → Streaming**.

### Stream Actor Spritesheet
**Event ID:** `EVENT_STREAM_ACTOR_SPRITESHEET`

Re-packs the sheet at build time, reserves the actor's band, and binds the actor to the streamed sheet at run time.

| Field | Type | Default | Description |
|---|---|---|---|
| Actor | Actor | Self | Actor that will stream its frames. |
| Sprite Sheet | Sprite | Last sprite | Sheet to stream. |
| Animation State | Animation state | Default | State to select on the streamed sheet. |
| Reserve tiles (0 = auto) | Number | 0 | Band size. 0 = the sheet's largest frame. |
| Upload first frame immediately | Checkbox | On | Waits for VBlank and uploads the current frame at once. |
| Apply sheet collision bounds | Checkbox | On | Copies the sheet's bounds onto the actor. |

### Stream Actor Spritesheet By Index
**Event ID:** `EVENT_STREAM_ACTOR_SPRITESHEET_BY_INDEX`

Same, for an actor selected by run-time index (0 = player). Does **not** reserve VRAM — pair it with *Reserve Streamed Actor Tiles*.

| Field | Type | Default | Description |
|---|---|---|---|
| Actor index | Value | 0 | Scene actor index, 0 = player. |
| Sprite Sheet | Sprite | Last sprite | Sheet to stream. |
| Animation State | Animation state | Default | State to select. |
| Band size limit (0 = auto) | Number | 0 | Never upload more than this many tiles. |
| Upload first frame immediately | Checkbox | On | |
| Apply sheet collision bounds | Checkbox | On | |

### Reserve Streamed Actor Tiles
**Event ID:** `EVENT_STREAM_ACTOR_RESERVE_TILES`

Build-time only, emits no bytecode. Gives an actor an exclusive sprite VRAM band and removes the streamed sheet (and the actor's own stub sheet, when nothing else needs it) from the scene's shared sprite pool.

| Field | Type | Default | Description |
|---|---|---|---|
| Actor | Actor | Self | Actor to reserve a band for. |
| Sprite Sheet | Sprite | Last sprite | Sheet used to size the band. |
| Reserve tiles (0 = from sheet) | Number | 0 | Explicit band size; use the largest frame of the biggest sheet the actor will ever stream. |

### Stop Streaming Actor
**Event ID:** `EVENT_STREAM_ACTOR_STOP` — releases the actor's slot. VRAM keeps the last streamed frame.

### Stop All Actor Streaming
**Event ID:** `EVENT_STREAM_ACTOR_STOP_ALL` — clears every slot.

### Upload Streamed Actor Frame
**Event ID:** `EVENT_STREAM_ACTOR_UPLOAD_NOW` — waits for VBlank and uploads the actor's current frame immediately, instead of waiting for the streamer. Costs one frame.

### Streamed Actor Set Enabled
**Event ID:** `EVENT_STREAM_ACTOR_SET_ENABLED` — suspends/resumes the whole streamer (engine field `streamable_actor_enabled`). Useful around code that needs the entire VBlank.

### Streamed Actor Set Tile Budget
**Event ID:** `EVENT_STREAM_ACTOR_SET_BUDGET` — changes the per-VBlank tile budget at run time (engine field `streamable_actor_budget`).

### Streamed Actor Get Info
**Event ID:** `EVENT_STREAM_ACTOR_GET_INFO` — writes *is streaming* (0/1), *band base tile* and *band size* into three variables.

---

## Inner Workings

### Build time: re-packing the sheet

`analyseStreamSheet()` in the event walks the precompiled sprite (`options.sprites`), which already holds the final optimised tile indices (`metasprites`) and the raw 2bpp tile bytes (`vramData`). For each metasprite it:

1. collects the distinct tiles the frame uses, in order of appearance, giving each a frame-local id (`0, 2, 4…` in 8×16 mode);
2. concatenates their bytes into one block, sharing the block with any earlier frame that produced identical bytes;
3. rewrites the metasprite entries to the local ids, clearing the `S_VRAM2` bit.

`writeAsset()` then emits `<sprite>_stream.c/.h` containing the packed tiles, a `stream_frame_t[]` table (`{ byte offset, tile count }` per frame), the rewritten metasprites, copies of the sheet's `animation_t[]` / `animations_lookup[]` / bounds, a `spritesheet_t` with a **null tileset**, and a `stream_sheet_t` descriptor.

The file name is derived from the sprite symbol, so several events referencing the same sheet emit identical content to the same file instead of duplicating it.

### Build time: VRAM accounting

Two things happen to the scene:

* `scene.actorsExclusiveLookup[actorId]` is set to the band size — the same map the stock *Set Actor Spritesheet* scan fills, which becomes `actor.reserve_tiles` in the generated scene data;
* the streamed sheet is spliced out of `scene.sprites`. GB Studio adds **any** sprite referenced by an event argument named `spriteSheetId` to the scene's shared VRAM pool, which for a streamed sheet would upload every tile it was supposed to avoid.

### Run time: the slot table

`vm_stream_actor()` copies the `stream_sheet_t` out of its bank, repoints `actor->sprite` at the generated sheet, loads its animations (and optionally bounds), and fills a `stream_slot_t`: actor pointer, expected sheet pointer, data bank, tile/frame pointers, the actor's `base_tile`, the band size and the resident frame. The VBlank handler is installed with `add_VBL()` on the first registration.

### Run time: staying in sync with OAM

The handler runs on *every* VBlank, but only some VBlanks are safe to upload in. `actor->frame` changes during the VM phase — a *Set Animation State*, *Set Animation Frame* or a state change inside a long event chain — and `actors_render()` may not run again for several frames while that script keeps the VM busy (a dialogue, for instance). Uploading a frame that has not been rendered yet puts the new tiles under the **old** metasprite for one frame, which reads as a sprite glitch.

GB Studio double buffers shadow OAM and flips `_shadow_OAM_base` exactly once per rendered frame (in `activate_shadow_OAM()`, including inside `ui.c`'s own text and menu loops). So the handler bails out unless that page changed since the last VBlank:

```
if _shadow_OAM_base == last seen page: return   // no render since last VBlank
remember the page
for each slot (round robin):
    skip if free, if actor->frame is already resident,
    if the actor no longer points at our sheet or moved band,
    or if the frame index is out of range
    read the frame descriptor from the data bank
    defer if the frame does not fit in the remaining budget
    set_sprite_data(base_tile, n_tiles, tiles + offset)
```

Because a render pass is always followed by `activate_shadow_OAM()` and then `wait_vbl_done()`, and nothing between them can touch `actor->frame`, a changed page guarantees `actor->frame` is exactly what the OAM about to be DMA'd was drawn from.

*Upload Streamed Actor Frame* and the **Upload first frame immediately** option deliberately bypass this — they are the "force it now" path, normally used at scene init behind a fade where the alternative is one frame of the placeholder sprite.

It saves and restores both the ROM bank and `VBK_REG`, so it can interrupt banked reads or a colour tile upload safely, and it never calls into the plugin's own ROM bank (the handler and the upload helper are `NONBANKED`, i.e. in HOME).

---

## Memory Footprint

Measured from the example project's link map (`STREAMABLE_ACTOR_SLOTS = 4`):

| Area | Cost |
|---|---|
| WRAM | 56 bytes — 2 engine fields, 4 × 13-byte slots, 2 bytes of state |
| SRAM | none |
| HOME (bank 0) | 89 bytes — the VBlank entry point (13 B) plus the two stubs that page in a sheet's data bank (37 B + 39 B) |
| Banked ROM | the streamer itself, the upload helper and the VM helpers, plus one re-packed tile block per distinct frame of each streamed sheet |

Bank 0 is the scarcest space in a GB Studio project, so the streamer runs from
the plugin's own switchable bank and keeps only those three stubs resident. For
context, the same code built `NONBANKED` needed 623 bytes there. See
[Bank 0 (HOME) Usage](#bank-0-home-usage).

Sprite VRAM saved per streamed actor is `(tiles of the whole sheet) − (tiles of its largest frame)`: in the example, 24 → 4 tiles for a 12-frame 16×16 sheet.

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
