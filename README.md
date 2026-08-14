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
5. [Compatibility](#compatibility)
6. [Events Reference](#events-reference)
7. [Engine Fields Reference](#engine-fields-reference)
8. [Media](#media)
9. [Memory Footprint](#memory-footprint)
10. [Bank 0 (HOME) Usage](#bank-0-home-usage)
11. [Changelog](#changelog)

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

1. Copy `src/StreamableActorPlugin` into your project's `plugins/` folder.

   The plugin replaces one stock engine file, `core/actor.c` — identical to the stock one apart from one include and one call at the top of `actors_render()` that brings streamed actors' tiles in before anything is drawn from them (it expands to nothing in VBlank mode). Its `order` is therefore `-1`, and it ships `engineAlt/` variants for **every** other engine plugin that also replaces `actor.c`, so it can be installed alongside any of them without either losing its changes. See [Compatibility](#compatibility).
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
| **Tile update mode** | VBlank mode | How a new frame reaches VRAM — see [Tile update modes](#tile-update-modes). |
| **Tiles uploaded per frame** | 4 | The per-VBlank upload budget, in 8×8 tiles. |
| **Streamed actor slots** | 4 | How many actors can stream at once. |

### Tile update modes

**VBlank mode** (default) copies each frame straight over the tiles the actor is drawing from. One band of VRAM per actor, but the whole frame has to land inside a single VBlank — so the tile budget has to be big enough for a whole frame, and a copy that outlasts VBlank both risks dropped tile bytes and holds off the LCD interrupt parallax scenes rely on.

On a **Game Boy Color** this mode moves the tiles with general purpose DMA instead: the hardware transfers 16 bytes in about 8 cycles, against roughly 208 for the same tile through the engine's byte-at-a-time copy. A four-tile frame goes from costing most of a VBlank to costing almost none of it, which is what makes the budget comfortable there. GDMA ignores the low four bits of its source address, so it is used only for sheets whose tile data landed 16-byte aligned — where it did not, the ordinary copy runs instead, slower but never wrong.

**VRAM buffer mode** does not touch VBlank at all. It reserves **two** bands per streamed actor and copies from the top of `actors_render()` — once per frame, before anything is drawn — into the band the actor is *not* drawing from, then switches the actor over to it. That timing gives it three properties VBlank mode cannot have:

- **Nothing is held off.** The copy is ordinary main-thread work, so the LCD interrupts that set parallax scroll and hide sprites behind the overlay run exactly when they should. This is the mode to use if either is flickering.
- **No budget, no latency.** The whole frame is copied in one go, and the frame it copies is the one the actor is about to be drawn with, so the tiles and the OAM entries pointing at them always agree.
- **Costs almost nothing when there is nothing to do.** The check runs once per frame over the streaming slots — not once per drawn actor — and which frame sits in each half of a band is remembered, so an unchanged or already-loaded frame is settled with a couple of byte compares in bank 0, without calling into the plugin bank at all.
- **Nothing half-drawn.** The LCD is mid-frame while the copy runs, which is exactly why the second band exists — the actor keeps drawing from the finished one until the copy completes.
- **Most frame changes copy nothing.** Frames drawn from byte-identical tiles share one block when the sheet is re-packed, so comparing block offsets is the same as comparing pixels — and both halves are compared. If the half being drawn from already holds them (a mirrored pose, a repeated step, a direction change that only flips the metasprite) nothing happens at all; if the *spare* half still holds them from earlier, the actor switches back to it without copying. A looping animation therefore settles into a ping-pong between the two halves and stops copying: across every animation in the example project, frame changes that copy fall by 74–82%.

The cost is double the sprite VRAM per streamed actor: a 4-tile frame reserves 8 tiles. The reservation is doubled for you at build time — no change to your events. This mode also replaces the engine's `actor.c` (see [Project Setup](#project-setup)).

---

## Size Limits and Restrictions

### The VBlank budget

Copying tiles takes time, and VBlank is short. The engine's guarded copy moves a tile in roughly 208 cycles — close to two scanlines — and a VBlank has about ten scanlines, some of which the engine has already spent by the time the streamer runs. **About 4 tiles fit on DMG, 8 at Game Boy Color double speed**, which is where the default comes from.

If several actors change frame at once and the budget runs out, the actors that did not fit keep showing their previous frame and are served **first** on the next frame — animation lags by a frame, nothing is corrupted.

A frame is never split, so the budget must be at least one frame of the sheet: a frame bigger than the whole budget is uploaded anyway, overrunning VBlank for that one actor rather than freezing its animation.

**This setting applies to VBlank mode only.** VRAM buffer mode copies outside VBlank and has nothing to budget.

### Parallax scenes

Interrupts stay disabled for the whole VBlank handler, so a copy that outlasts VBlank holds off every LCD interrupt waiting behind it — the parallax row scroll scheduled for `LY = 0`, and the sprite hiding the overlay handler does around the window. The background flickers along its top rows and the sprites flicker with it, on exactly the frames where a streamed actor's tiles changed.

Two things address it:

- **Keep the budget within a VBlank** — about 4 tiles on DMG, 8 at Game Boy Color double speed. In VBlank mode this is the only lever, and it has a floor: the budget cannot go below one frame of the sheet, because a frame is never split. Several actors animating together, or frames larger than about 4 tiles, will still overrun.
- **Use VRAM buffer mode**, which removes the cause rather than shrinking it: it copies from the render loop instead of the VBlank handler, so nothing is ever held off. The cost is double the sprite VRAM per streamed actor and a replaced `actor.c`.

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

## Compatibility

`core/actor.c` is the only stock engine file this plugin replaces, which sets its `order` to `-1`. Five other engine plugins replace it too:

| Plugin | order |
|---|---|
| ContinuousScenePlugin | −13 |
| ScreenScrollPlugin | −13 |
| DynamicActorPlugin | −7 |
| SpritesheetChangeBufferPlugin | −4 |
| EditActorActiveIndexPlugin | −1 |

GB Studio applies engine plugins in ascending `order`, so the last one to load is the one whose `actor.c` survives. At `-1` — and winning the tie with EditActorActiveIndexPlugin alphabetically — this plugin loads last of the six, so **it hosts the merged copies and none of the other five need changing**.

`engineAlt/` therefore carries all **23** reachable combinations: any one of ContinuousScene or ScreenScroll (they are alternatives and never appear together), with any mix of DynamicActor, SpritesheetChangeBuffer and EditActorActiveIndex. Each is that combination's own merged `actor.c` with this plugin's hooks re-applied.

Plugins that do not replace `actor.c` — MetaTile, SceneStackEx, FadeStreet and the rest — need no variant and no rule: a rule matches on the plugins it lists being present, so extra plugins alongside them do not affect the match.

### Regenerating the variants

`engineAlt/` is generated, never hand-merged:

```bash
node tools/gen_enginealt.js
```

It walks the 23 combinations, picks the `actor.c` that would win without this plugin (from the highest-order plugin in that combination, using *its* merged copy when others are present), and re-applies the same edits: one include and one call at the top of `actors_render()`. Every version of that function opens the same way, so a single anchor covers all of them regardless of whether they draw the player separately or fold it into the actor loop. Anything unexpected in a base file aborts the run rather than producing a silent mis-merge.

Re-run it whenever one of those five plugins changes its `actor.c`, then run the patch builder to produce the distributed `patched/` form and the `engineAltRules`.

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

`StreamableActorExample/` is a working project: the player, Link and an NPC are placed with a 4-tile stub sprite and stream their sheets into 4-tile bands, so the scene's shared sprite pool ends up empty and the three actors together use 12 sprite tiles. It ships in VBlank mode; switching **Tile update mode** to *VRAM buffer mode* in the engine settings is the only change needed to try the other one, and takes the three actors to 24 tiles.

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
| Tile update mode → *VRAM buffer mode* | +153 B | +20 B | +280 B |
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

Measured against the stock GB Studio **4.3.0-e1** engine by `measure_plugin_memory.js` (per-file SDCC compile with GB Studio's own build flags, at default engine settings; report of 2026-08-13). Figures are this plugin's *delta* versus stock — a file that replaces a stock engine file counts only the difference, which is why a plugin can come out negative. Using the plugin's events additionally compiles a few bytes of GBVM script per call into your project's script banks, on top of the fixed cost below.

| Budget | Cost |
|---|---|
| Bank 0 (HOME) | +161 bytes |
| WRAM | +58 bytes |
| Banked ROM | +2,149 bytes |

- **Bank 0:** 161 bytes are resident in the non-switchable bank (`streamable_actor.c`) — the VBlank handler and the two routines that page in a sheet's data bank; everything else lives in a switchable bank. The handler decides for itself whether there is anything to copy, which is what keeps an idle VBlank off the plugin bank entirely. See [Bank 0 (HOME) Usage](#bank-0-home-usage).
- **WRAM:** 58 bytes at the default 4 streaming slots — 13 bytes per slot plus the engine fields and state. Fewer slots cost proportionally less.
- **Banked ROM:** 2,149 bytes for the streamer, the upload helper and the VM helpers. Each streamed sheet also compiles one re-packed tile block per distinct frame, on top of that figure.
- **Sprite VRAM saved** per streamed actor is the whole sheet's tile count minus its largest frame's tile count — in the example, 24 tiles down to 4 for a 12-frame 16×16 sheet.
- **Engine WRAM headroom:** a stock GB Studio 4.3.0 project leaves about **854 bytes** of WRAM free (usable engine WRAM is 7,776 bytes at 0xC0A0–0xDF00; the stock engine uses 6,922). With this plugin installed roughly **796 bytes** remain. That does not change with the number of global variables your project defines: the script memory array is a fixed 3,584 bytes at stock engine settings (VM_HEAP_SIZE + VM_MAX_CONTEXTS × VM_CONTEXT_STACK_SIZE = 768 + 16 × 64 words).
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
| Bank 0 used by this plugin | **+161** (VBlank mode) / **+314** (VRAM buffer mode) |
| Bank 0 free with this plugin installed | **1,290** of 16,384 (92% used), or **1,137** in VRAM buffer mode |

Everything else this plugin adds lives in banked ROM.

| Module | This plugin | Stock engine | Bank 0 cost |
|---|---|---|---|
| `core/streamable_actor.c` | 161 | — *(new file)* | +161 |

Modules that replace or patch a stock engine file only cost the *difference*:
the stock version's bank 0 bytes were being spent anyway.

<details><summary>How this was measured</summary>

GB Studio 4.3.0-e1, default engine settings. Each module is compiled with the
toolchain and flags GB Studio itself uses, and the `A _HOME size` record SDCC
writes into the resulting `.rel` object is read back; the stock column is the
same compile of the engine file this module replaces.

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

### 2026-08-13

- New **Tile update mode** setting. *VRAM buffer mode* copies from the render loop rather than the VBlank handler, into a second band the actor is not drawing from, then switches the actor over to it. Nothing is held off, so the LCD interrupts that set parallax scroll and hide sprites behind the overlay keep their timing — which is what makes the background and the sprites flicker when the VBlank handler runs long. It costs double the sprite VRAM per streamed actor and replaces the engine's `actor.c`. *VBlank mode* is the default and behaves exactly as before.
- VBlank mode now uses general purpose DMA on a Game Boy Color, about 8 cycles per tile instead of 208. It applies to sheets whose tile data is 16-byte aligned, since GDMA ignores the low four bits of its source address; the ordinary copy still runs for the rest.
- VBlank mode's handler now decides in bank 0 whether there is anything to copy — the render gate, the budget check and a look over the slots — and only calls into the plugin bank when a streamed actor has actually changed frame. A VBlank with nothing to do costs about 84 cycles instead of a banked call into a function that was going to return anyway, which is VBlank time given back to the rest of the engine.
- VRAM buffer mode now checks its slots **once per frame** at the top of `actors_render()` rather than once per drawn actor, and remembers which frame sits in each half of a band, so a frame that is already loaded is settled with byte compares in bank 0 instead of a banked call and a bank-switched descriptor read.
- VRAM buffer mode now skips the copy when the tiles it would write are already in VRAM. Both halves of the band are checked: the half being drawn from (nothing happens at all) and the spare half (switch back to it without copying), so a looping animation ping-pongs between the two and stops copying. Measured across the example project's animations, frame changes that copy fall by 74–82%.
- Fixed *Upload Streamed Actor Frame* writing to the wrong half of the band in VRAM buffer mode once an actor had been switched over to the spare one.
- The plugin now replaces `core/actor.c`, so it declares `order: -1` and ships `engineAlt/` variants for all 23 combinations of the five other engine plugins that replace the same file. It loads last of the six, so none of those plugins need changes to sit alongside it.
- **Tiles uploaded per frame** now defaults to 4 rather than 8. Only about 4 tiles actually fit in a DMG VBlank, so the old default overran it whenever two actors changed frame together — which showed up as the background and sprites flickering.

### 2026-08-02

- Initial release: LADX-style per-frame sprite streaming into a small reserved VRAM band.
