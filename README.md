# gbs-StreamableActorPlugin

**Version 4.3.0. Requires GB Studio 4.3.0 or newer.**

Loads an actor's animation one frame at a time, so a sprite sheet costs the space of its **largest single frame** rather than all of its frames put together. The number of frames, directions and animation states stops mattering.

That is what lets a character have forty animations on a machine with room for about six. In the example project, a Link sprite of 102 tiles, more than the hardware can hold at once, runs in 4 tiles.

Frames can have completely different shapes and tile counts. Whatever the current frame needs is what gets loaded.


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
8. [FAQ](#faq)
9. [Media](#media)
10. [Memory Footprint](#memory-footprint)
11. [Bank 0 (HOME) Usage](#bank-0-home-usage)
12. [Changelog](#changelog)

---

## Concepts

### How GB Studio normally spends sprite VRAM

There is room for 256 sprite tiles, of which 128 are shared with background tiles, or 192 with GB Studio's interface tiles, and double that in colour-only mode. When a scene loads, GB Studio uploads **every tile of every sprite sheet the scene uses**. A 24-frame player sheet with 40 unique tiles holds 40 slots for the whole scene, even though only 4 to 8 of them are on screen at any moment.

### What this plugin does

- **During the build** it repacks the sheet so each frame owns its own run of tiles. Identical runs are shared, so a repeated frame costs nothing extra.
- **When the scene loads** the actor gets a private block of tile slots exactly as big as the sheet's largest frame, and the sheet itself is kept out of the scene's shared pool, so nothing is uploaded for it.
- **While the game runs** the plugin notices the actor's frame change and copies that frame's tiles into the block. Nothing about how the actor is drawn changes, so every stock animation event keeps working.

The copy is timed so the new tiles and the new sprite layout appear on the same displayed frame, with no mismatch between them.

---

## Project Setup

1. Copy `src/StreamableActorPlugin` into your project's `plugins` folder.

   The plugin replaces one stock engine file, the actor handling code. Its copy is the stock one plus a single call that brings streamed tiles in before anything is drawn from them, and that call does nothing in VBlank mode. Compatibility variants ship for every other plugin that replaces the same file. See [Compatibility](#compatibility).
2. **Give the actor a small placeholder sprite in the editor.** That sheet is still loaded when the scene starts, so keep it tiny. A single 16 by 16 frame is 4 tiles. It is only visible until the first real frame arrives.
3. Add **Stream Actor Spritesheet** to the scene's **On Init**, or the actor's, and pick the actor and the real sheet.
4. Build. The actor now animates normally with every stock event, including **Set Animation State**, **Set Animation Frame**, movement and direction changes, and the plugin follows whatever changes the frame.

For an actor whose number is only known while the game runs, use **Reserve Streamed Actor Tiles** during the build, picking the actor, together with **Stream Actor Spritesheet By Index** at run time.

---

## Engine Settings

Found under **Settings**, then **Engine**, then **Streamable Actors**.

| Setting | Default | Description |
|---|---|---|
| **Streaming enabled** | on | Master switch for the streamer. |
| **Tile update mode** | VBlank mode | How a new frame gets loaded. See [Tile update modes](#tile-update-modes). |
| **Tiles uploaded per frame** | 4 | How many tiles may be copied per screen blank. |
| **Streamed actor slots** | 4 | How many actors can stream at once. |
| **Use HDMA on Game Boy Color** *(VBlank mode only)* | on | Copy tiles with the fast hardware transfer. See [Tile update modes](#tile-update-modes). |
| **Force aligned tile pools** *(VBlank mode, with HDMA on)* | off | Guarantees the fast transfer works, at the cost of ROM. See [The tile pool](#the-tile-pool). |

### Tile update modes

**VBlank mode**, the default, copies each frame straight over the tiles the actor is drawing from. It uses one block of slots per actor, but the whole frame has to be copied during the screen blank, so the tile budget has to cover a whole frame. A copy that runs past the blank risks losing bytes and delays the interrupt that parallax scenes rely on.

On **Game Boy Color** this mode uses the hardware's fast transfer instead. It moves a tile in about 8 cycles against roughly 208 for the software copy, so a four-tile frame goes from most of a screen blank to almost none of it. That is the **Use HDMA on Game Boy Color** setting, on by default.

The fast transfer needs its source to start on a 16-byte boundary. See [The tile pool](#the-tile-pool). Nothing checks that at run time, deliberately: **if your streamed actors draw scrambled tiles on a Game Boy Color, the pool did not land on a boundary.** Turn on [Force aligned tile pools](#force-aligned-tile-pools). Turning **Use HDMA on Game Boy Color** off instead falls back to the software copy, which is correct wherever the pool lands and only slower.

**VRAM buffer mode** stays out of the screen blank entirely. It reserves **two** blocks per streamed actor and copies during the ordinary game loop, once per frame and just before drawing, into the block the actor is *not* using, then switches the actor over. That timing gives it five properties VBlank mode cannot have:

- **Nothing is delayed.** The copy is ordinary work, so the interrupts that set parallax scroll and hide sprites behind the overlay run exactly on time. Use this mode if either is flickering.
- **No budget and no lag.** The whole frame is copied at once, and it is the frame the actor is about to be drawn with, so the tiles and the sprite always agree.
- **It costs almost nothing when there is nothing to do.** The check runs once per frame over the streaming slots rather than once per actor, and which frame sits in each half of a block is remembered. An unchanged or already loaded frame is settled with a couple of comparisons, about 48 cycles, or 0.3% of a frame.
- **Nothing is caught half drawn.** The screen is mid-frame while the copy runs, which is exactly why the second block exists. The actor keeps drawing from the finished one until the copy completes.
- **Most frame changes copy nothing.** Frames made of identical tiles share one run when the sheet is repacked, so comparing runs is the same as comparing pixels, and both halves are compared. A mirrored pose, a repeated step or a direction change that only flips the sprite needs no copy at all, and a frame still sitting in the spare half is switched back to without copying. A looping animation settles into alternating between the two halves and stops copying: across every animation in the example project, frame changes that copy fall by 74 to 82%.

The cost is twice the sprite tiles per streamed actor, so a 4-tile frame reserves 8. The build doubles the reservation for you, with no change to your events. This mode also replaces the stock actor handling code. See [Project Setup](#project-setup).

---

## Size Limits and Restrictions

### The VBlank budget

Copying tiles takes time and the screen blank is short. The software copy moves a tile in roughly 208 cycles, close to two screen lines, and a blank is about ten lines, some of which the engine has already spent. **About 4 tiles fit on original Game Boy and 8 at Game Boy Color's faster speed**, which is where the default comes from.

When several actors change frame at once and the budget runs out, the ones that did not fit keep their previous frame and go first next time. The animation lags by a frame and nothing is corrupted.

A frame is never split, so the budget has to be at least one frame of the sheet. A frame larger than the whole budget is copied anyway, running past the blank for that one actor rather than freezing its animation.

**This setting applies to VBlank mode only.** VRAM buffer mode copies elsewhere and has nothing to budget.

### Parallax scenes

A copy that runs past the screen blank delays the interrupts waiting behind it: the parallax row scroll set for the top of the screen, and the sprite hiding done around the overlay. The top rows of the background flicker and the sprites flicker with them, on exactly the frames where a streamed actor's tiles changed.

Two ways to deal with it:

- **Keep the budget inside a blank**, about 4 tiles on original Game Boy and 8 at Game Boy Color's faster speed. In VBlank mode this is the only control, and it has a floor, because the budget cannot go below one frame of the sheet. Several actors animating together, or frames larger than about 4 tiles, still overrun.
- **Use VRAM buffer mode**, which removes the cause. It copies during the game loop rather than the screen blank, so nothing is ever delayed. The cost is twice the sprite tiles per streamed actor and a replaced actor handling file.

### The actor's editor spritesheet still costs VRAM

The sheet you picked in the editor is uploaded into the reserved block before any script runs. A sheet bigger than the block makes the block grow to fit it, which undoes the saving. Always give a streamed actor a tiny placeholder sprite.

### The tile pool

Every streamed sheet in the project puts its frame data into one shared pool, built when the project is built. Each sheet points at its own part of it. When the next sheet will not fit in a bank, a second pool starts, and a later small sheet fills whichever one still has room.

This exists because of the fast Game Boy Color transfer, which needs its source to start on a 16-byte boundary, and nothing in the build tools can ask for one. Where the data lands is decided by the bank packer.

Pooling turns one gamble per sheet into one gamble for all of them. Every part of a pool is a whole number of tiles, so either all of them are on a boundary or none are, and a pool is a large single block that the packer usually has to give a fresh bank, which starts on a boundary. In the example project the pool is 5,408 bytes across three sheets and lands at the start of its bank, so all three get the fast transfer. Before pooling, only one of the three did.

It is a strong bias rather than a guarantee, which is why a badly placed pool is left to show itself as scrambled tiles rather than being quietly worked around.

#### Force aligned tile pools

Turning this setting on pads every tile pool out to a full 16 KB bank. A pool that size can only go in an empty bank, which it then fills, so nothing can sit in front of it and it always starts on a boundary. That is the only way to guarantee the fast transfer, because every other route is closed off by the build tools.

The cost is the padding, up to 16 KB per pool. In the example project a pool of 5,408 bytes is padded by 10,976 and takes a bank to itself. The ROM file does not grow, because what it displaced moves into banks that were empty. In a project with no spare banks it will grow the cartridge.

Leave it off unless you are building for Game Boy Color, have seen the scrambled tiles that mean the pool landed badly, and have the ROM to spare. Pools usually land correctly without it. It does nothing on original Game Boy builds, where there is no fast transfer.

It applies only in **VBlank mode with HDMA on**, since that is the only case that needs the boundary. It is hidden otherwise, and a leftover value from before you changed the mode is ignored rather than quietly spending ROM.

### ROM size

A frame's tiles are kept together, so tiles cannot be shared between frames. A sheet whose frames overlap heavily, such as a large sprite where only a hand moves, grows in ROM. Identical frames still share one run, and a sheet with little repetition between frames costs the same as before. In the example project both sprites produce exactly as many tile bytes as their normal tilesets while dropping from 24 sprite tiles to 4.

If nothing else in the project uses the streamed sheet, GB Studio still builds its normal tileset as well. Streamed actors do not use that copy.

### 8×16 sprite mode

In 8 by 16 mode tiles come in pairs, so a block is always an even number of tiles. There is nothing to configure.

### Colour sprites

Colour-only sheets normally split their tiles across both tile banks. Streamed frames always go in the first, since a streaming block is small enough not to need the second.

### Scene changes

A slot is only used while the actor still points at the streamed sheet and still owns the same block of tiles. Registrations left over from a previous scene are ignored and their slots reused, rather than writing into another actor's tiles. **Stop All Actor Streaming** frees them immediately if you want.

### Save states

Streaming is not part of a save. Run the **Stream Actor Spritesheet** events again after a load. They are normally in **On Init**, which runs anyway.

### Stock spritesheet events on a streamed actor

The stock **Set Actor Spritesheet** event pointed at a streamed sheet uploads nothing. Use **Stream Actor Spritesheet** again to move a streamed actor onto another streamed sheet. The stock event with an ordinary sheet stops the streaming for you, but the block is only as big as it was reserved.

---

## Compatibility

The actor handling code is the only stock engine file this plugin replaces. Five other plugins replace it too: ContinuousScene, ScreenScroll, DynamicActor, SpritesheetChangeBuffer and EditActorActiveIndex.

GB Studio loads plugins in a set order, and the last one to load is the one whose copy survives. This plugin loads last of the six, so **it carries the merged copies and none of the other five need changing**.

It ships all **23** reachable combinations: either ContinuousScene or ScreenScroll, which never appear together, with any mix of DynamicActor, SpritesheetChangeBuffer and EditActorActiveIndex. Each is that combination's own merged copy with this plugin's changes reapplied.

Plugins that do not replace the actor handling code, such as MetaTile, SceneStackEx and FadeStreet, need no variant and do not affect the match.

---

## Events Reference

All events live under **Actor**, in the **Streaming** section.

---

### Stream Actor Spritesheet

Repacks the sheet during the build, reserves the actor's tile block, and points the actor at the streamed sheet.

| Field | Default | Description |
|---|---|---|
| Actor | Self | Actor that will stream its frames. |
| Sprite Sheet | Last sprite | Sheet to stream. |
| Animation State | Default | State to select on the streamed sheet. |
| Reserve tiles (0 = auto) | 0 | Block size. 0 uses the sheet's largest frame. |
| Upload first frame immediately | On | Loads the current frame straight away rather than waiting. |
| Apply sheet collision bounds | On | Copies the sheet's bounds onto the actor. |

---

### Stream Actor Spritesheet By Index

The same, for an actor chosen by number while the game runs, where 0 is the player. It does **not** reserve tiles, so pair it with **Reserve Streamed Actor Tiles**.

| Field | Default | Description |
|---|---|---|
| Actor index | 0 | Which actor. 0 is the player. |
| Sprite Sheet | Last sprite | Sheet to stream. |
| Animation State | Default | State to select. |
| Band size limit (0 = auto) | 0 | Never load more than this many tiles. |
| Upload first frame immediately | On | Loads the current frame straight away rather than waiting. |
| Apply sheet collision bounds | On | Copies the sheet's bounds onto the actor. |

---

### Reserve Streamed Actor Tiles

Runs during the build and adds no code to your game. Gives an actor a private block of sprite tiles and takes the streamed sheet out of the scene's shared pool, along with the actor's placeholder sheet when nothing else needs it.

| Field | Default | Description |
|---|---|---|
| Actor | Self | Actor to reserve tiles for. |
| Sprite Sheet | Last sprite | The sheet whose size decides the block. |
| Reserve tiles (0 = from sheet) | 0 | A size of your own. Use the largest frame of the biggest sheet the actor will ever stream. |

---

### Stop Streaming Actor

Frees the actor's streaming slot. The last loaded frame stays on screen.

---

### Stop All Actor Streaming

Clears every streaming slot.

---

### Upload Streamed Actor Frame

Loads the actor's current frame straight away rather than waiting for the next update. Costs one frame.

---

### Streamed Actor Set Enabled

Pauses or resumes all streaming. Useful around code that needs the whole screen blank to itself.

---

### Streamed Actor Set Tile Budget

Changes the tile budget while the game runs.

---

### Streamed Actor Get Info

Writes whether the actor is streaming, its first tile and its block size into three variables.

---

## Engine Fields Reference

These sit behind the two settings events and can also be read or written with **Engine Field Value**.

| Field | Description |
|---|---|
| **Streaming enabled** | Whether streaming is currently running. |
| **Tiles uploaded per frame** | The tile budget per screen blank. |

**Tile update mode**, **Streamed actor slots** and **Force aligned tile pools** are decided when the project is built and cannot be changed while it runs.

---

## FAQ

**My character has too many animations to fit in sprite memory. Does this fix it?**
Yes, and that is the point. Only the current frame is loaded, so a sheet with forty animations
costs the same as its largest single frame. The example runs a 102-tile Link sprite in 4 tiles.

**Do I have to change how my animations work?**
No. Every stock event keeps working, including **Set Animation State**, **Set Animation Frame**,
movement and direction changes. The plugin follows whatever changes the frame.

**Why does my actor need a placeholder sprite?**
The sheet chosen in the editor is still loaded when the scene starts, and the reserved block grows
to fit it. Use a single 16 by 16 frame, which is 4 tiles, so the block stays small.

**How much space does a streamed actor use?**
Its largest single frame, or twice that in VRAM buffer mode. A four-tile frame is 4 tiles, or 8 in
buffer mode.

**My parallax background flickers along the top when the actor animates.**
The tile copy is running past the screen blank and delaying the interrupt. Switch **Tile update
mode** to **VRAM buffer mode**, which copies elsewhere entirely, or lower the tile budget.

**My streamed actors show scrambled tiles on Game Boy Color.**
The tile pool did not land on the boundary the fast transfer needs. Turn on **Force aligned tile
pools**, or turn off **Use HDMA on Game Boy Color** to fall back to the slower copy.

**My animation lags by a frame when several actors move at once.**
The tile budget ran out. Raise **Tiles uploaded per frame** if you have blank time to spare, or
switch to VRAM buffer mode, which has no budget.

**Which update mode should I use?**
VBlank mode by default, since it uses half the tiles. Switch to VRAM buffer mode if you have
parallax or overlay flicker, several actors animating at once, or frames larger than about 4 tiles.

**How many actors can stream at once?**
Four by default, and up to 16 with the **Streamed actor slots** setting. Each slot costs 14 bytes
of memory.

**Did my ROM get bigger?**
Only for a sheet whose frames overlap heavily, such as a large sprite where one hand moves,
because tiles cannot be shared between frames. Identical frames still share one run.

**Can I stream an actor spawned while the game runs?**
Yes. Use **Reserve Streamed Actor Tiles** during the build to set the block aside, and **Stream
Actor Spritesheet By Index** when you know which actor it is.

**Does streaming survive a save and load?**
No. Run the **Stream Actor Spritesheet** events again after a load. They are usually in **On Init**,
which runs anyway.

**Does it work with the DynamicActor, SpritesheetChangeBuffer or ContinuousScene plugins?**
Yes, in any combination. Compatibility variants for all 23 reachable combinations ship with it.

---

## Media

`StreamableActorExample/` is a working project. The player, Link and an NPC each have a 4-tile placeholder sprite and stream their real sheets into 4-tile blocks, so the scene's shared pool ends up empty and the three actors together use 12 sprite tiles. It ships in VBlank mode. Switching **Tile update mode** to **VRAM buffer mode** is the only change needed to try the other one, and takes the three actors to 24 tiles.

The Link actor uses his real tile sheet and animation table, imported from the [LADX disassembly](https://github.com/zladx/LADX-Disassembly). Press **A** and **B** to step through all 41 imported actions, including walk, push, shield, lift, pull, swim, jump and falling, with a dialogue naming each one.

That sheet is 102 unique tiles, more than the hardware can hold at once, so the normal loader cannot fit it at all. Streamed, Link costs **4 tiles**, and frames whose two halves share a tile pair load only 2.

> The imported graphics are Nintendo's and are not checked in. `tools/import_ladx_link.js` and `tools/gen_link_demo_scene.js` rebuild them from your own disassembly checkout.

---

<!-- SETTINGCOST:BEGIN -->
### What each engine setting costs

Each setting changes what gets compiled. Figures are what you **get back by turning
the setting off**. Rows marked *off by default* show what turning it **on** costs, and
sliders show the cost per step. "none" means that budget does not move.

| Setting | Bank 0 | WRAM | Banked ROM |
|---|---|---|---|
| Tile update mode → *VRAM buffer mode* | -124 B | +20 B | +537 B |
| Streamed actor slots *(slider 1–16, default 4)* | none | 14 B/step | none |
| Use HDMA on Game Boy Color | none | none | none |
| Force aligned tile pools *(off by default, so this is the cost of turning it on)* | none | none | none |

- **Streamed actor slots**: going from 1 to 16 moves WRAM by +210 B.

- **Use HDMA on Game Boy Color** only applies when *Tile update mode* is enabled.
- **Force aligned tile pools** only applies when *Tile update mode* and *Use HDMA on Game Boy Color* are enabled.

<details><summary>How these were measured</summary>

GB Studio 4.3.0-e1. This plugin's engine code was compiled with the toolchain and
flags GB Studio itself uses, and the size of each part of the result was read back and
sorted into the three budgets: the fixed bank 0, work RAM, and switchable ROM banks.

Two caveats. Only this plugin's own engine sources are measured, so a setting that also
changes a shared data structure can move a few more bytes elsewhere. And each setting is
toggled on its own, so a few measure slightly *negative* when enabling their code lets
the compiler drop a fallback path, and a setting that gates other settings shows only
its own contribution.

</details>
<!-- SETTINGCOST:END -->

Two rows read as free above but are not, because the measurement builds for original Game Boy and counts engine code only:

- **Use HDMA on Game Boy Color** only exists in a colour build, so a monochrome measurement sees nothing either way. In a colour build it replaces the software copy with the hardware transfer: slightly less bank 0 code, and far less screen blank time.
- **Force aligned tile pools** adds no code at all, which is why it reads as free, but it adds up to 16 KB of ROM data per tile pool. See [The tile pool](#the-tile-pool).

## Memory Footprint

Measured against the stock GB Studio **4.3.0-e1** engine at default engine settings, report of 2026-08-13. Figures are the difference against a stock project: a file that replaces a stock engine file counts only the change, which is why a plugin can come out negative. Each event you use also compiles a few bytes of script into your project, on top of the fixed cost below.

| Budget | Cost |
|---|---|
| Bank 0 (HOME) | +161 bytes |
| WRAM | +62 bytes |
| Banked ROM | +2,173 bytes |

- **Bank 0:** 161 bytes sit in the fixed bank: the screen blank handler and the two routines that reach a sheet's data. Everything else lives in a switchable bank. The handler works out for itself whether there is anything to copy, which is what keeps an idle blank off the plugin's bank entirely. See [Bank 0 (HOME) Usage](#bank-0-home-usage).
- **WRAM:** 62 bytes at the default 4 slots, which is 14 per slot plus the settings and state. Fewer slots cost proportionally less.
- **Banked ROM:** 2,173 bytes for the streaming code. Each streamed sheet also builds one repacked run of tiles per distinct frame, on top of that.
- **Sprite tiles saved** per streamed actor is the whole sheet's tile count minus its largest frame's. In the example that is 24 tiles down to 4 for a twelve-frame 16 by 16 sheet.
- **Engine WRAM headroom:** a stock GB Studio 4.3.0 project leaves about **854 bytes** of WRAM free (the engine has 7,776 bytes to work with and uses 6,922 of them). With this plugin installed roughly **792 bytes** remain. Adding more global variables to your project does not change that figure, because script memory is a fixed 3,584 byte block at stock engine settings.
- **SRAM:** not used.

---

<!-- BANK0:BEGIN -->
## Bank 0 (HOME) Usage

Bank 0 is the 16 KB fixed ROM bank shared by the GB Studio engine core, the
interrupt handlers and the GBDK runtime. Extra banked ROM is cheap to add,
bank 0 is not, so bank 0 is usually the first thing a project runs out of.

| | Bytes |
|---|---|
| Bank 0 used by this plugin | **+161** (VBlank mode) / **+37** (VRAM buffer mode) |
| Bank 0 free with this plugin installed | **1,290** of 16,384 (92% used), or **1,414** in VRAM buffer mode |

Everything else this plugin adds lives in banked ROM.

| Module | This plugin | Stock engine | Bank 0 cost |
|---|---|---|---|
| `core/streamable_actor.c` | 161 | none *(new file)* | +161 |

A module that replaces a stock engine file costs only the *difference*, because
the stock version's bank 0 bytes were being spent anyway.

<details><summary>How this was measured</summary>

GB Studio 4.3.0-e1, default engine settings. Each module was compiled with the
toolchain and flags GB Studio itself uses, and the bank 0 size the compiler
recorded was read back. The stock column is the same compile of the engine file
the module replaces.

The "free" figure assumes a stock project with this plugin and nothing else.
Your own number will differ, because other plugins and any engine settings that
change what the core compiles move it too.

</details>
<!-- BANK0:END -->

## Changelog

Grouped by the date each change was merged into the official
[gb-studio-plugins](https://github.com/gb-studio-dev/gb-studio-plugins) repository.

Only bug fixes, new features and feature changes are listed. Engine version
bumps, patch regeneration, packaging fixes and documentation edits are omitted.

### 2026-08-14

- VRAM buffer mode's per-frame check moved slightly later in the game loop, where the tiles still land before anything is drawn from them, and out of bank 0. That hands **277 bytes of bank 0 back to the project**, about a third of what a stock project has spare, and costs about 48 cycles per frame instead of 6, which is 0.24% of a frame.
- Added **Use HDMA on Game Boy Color**, on by default. The fast hardware copy is now used without checking at run time whether the tile pool sits on the right boundary, so a pool that does not shows up as scrambled tiles. That is the cue to turn **Force aligned tile pools** on. Turning HDMA off falls back to the slow copy, which is correct either way.
- Added **Force aligned tile pools**, off by default and shown only in VBlank mode with HDMA on. It pads each tile pool out to a whole ROM bank, which forces the packer to give it a bank of its own and guarantees the boundary the fast copy needs. Costs up to 16 KB of ROM per pool.
- Streamed tile data is now gathered into shared pools instead of one block per sheet, starting a new pool whenever the next sheet will not fit in a bank. Every sheet in a pool shares its placement, so either all of them sit on the boundary the fast copy needs or none do, and a pool is large enough that the packer usually gives it a fresh bank, which does. In the example project all three sheets now qualify where only one did before.

### 2026-08-13

- Added the **Tile update mode** setting. **VRAM buffer mode** copies during the game loop rather than the screen blank, into a second block the actor is not drawing from, then switches the actor over. Nothing is delayed, so the interrupts that set parallax scroll and hide sprites behind the overlay keep their timing, which is what makes the background and sprites flicker when the blank handler runs long. It costs twice the sprite tiles per streamed actor and replaces the stock actor handling file. **VBlank mode** is the default and behaves as before.
- VBlank mode now uses the hardware transfer on Game Boy Color, about 8 cycles per tile instead of 208. It applies to sheets whose tile data sits on the boundary that transfer needs, and the ordinary copy still runs for the rest.
- VBlank mode's handler now decides in bank 0 whether there is anything to copy, checking the budget and looking over the slots, and only reaches into the plugin's bank when a streamed actor has actually changed frame. A blank with nothing to do costs about 84 cycles instead of a bank switch into a function that was going to return anyway.
- VRAM buffer mode now checks its slots **once per frame** rather than once per drawn actor, and remembers which frame sits in each half of a block, so a frame already loaded is settled with a couple of comparisons rather than a bank switch.
- VRAM buffer mode now skips the copy when the tiles are already loaded. Both halves of the block are checked, the one being drawn from and the spare, so a looping animation alternates between the two and stops copying. Across the example project's animations, frame changes that copy fall by 74 to 82%.
- Fixed **Upload Streamed Actor Frame** writing to the wrong half of the block in VRAM buffer mode once an actor had switched to the spare one.
- The plugin now replaces the stock actor handling file, and ships compatibility variants for all 23 combinations of the five other plugins that replace it. It loads last of the six, so none of them need changes.
- **Tiles uploaded per frame** now defaults to 4 rather than 8. Only about 4 fit in an original Game Boy screen blank, so the old default overran it whenever two actors changed frame together, which showed up as flickering.

### 2026-08-02

- Initial release: per-frame sprite loading into a small reserved block of tiles, in the style of Link's Awakening.
