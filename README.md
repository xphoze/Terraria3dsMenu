# Terraria3dsMenu
a mod menu for o3ds and n3ds terraria!

so i started a project early morning 10/10 wanting to add mods similar to tmodloader to add either quality of life or new features that didnt make the final cut for the 3ds. during that whole process i asked claude sonnet 5.5 to reverse engineer my dumped terraria files and it happened to find its own compiled in dev settings on the retail version that wasnt accessible at first. now this list is gonna be pretty long and i told claude to explain what happened here. so heres claudeeeee

FYI: please tell me what needs to be fixed or added and ill do it asap 🙏



The author gave me a dump of their own copy of the game and asked me to reverse engineer it so they could add
mods. Early on, a search of the game's text turned up something that shouldn't be in a shipped game: **Re-Logic's own developer
console and developer settings screen, still compiled into the retail 3DS build.** Nothing in the game ever opens them, so players
have never seen them.

From there it went in stages:

1. **Found the dev console.** About 87 commands (`item`, `boss`, `tp`, `godmode`, `spawnhouse`, ...), each with its original on-screen
   message. Opening it only needs one flag in the game's state manager to be set, so I hooked the main update loop and set it on a
   button combo.
2. **Found the dev settings screen.** A scrolling list of seven original developer options (floating damage text, a precision cursor
   mode, copy/load saves, force save, go to main menu, a Nintendo networking test). It's registered like every other screen, so it can
   be opened by name.
3. **Replaced its contents with a real menu.** I swapped in my own button list, built one screen per folder, and hooked the button
   handler so each button runs a command.
4. **Fixed it on real hardware.** The first run on a 3DS crashed: the game's input system wasn't ready on the very first frames.
   Luma's crash dump pinpointed the exact instruction, so the fix was a one-line readiness check.

The how and why of every step, including each bug, is in [HOW_IT_WORKS.md](HOW_IT_WORKS.md).

## Controls

| Combo | Action |
|---|---|
| **L + R + D-pad Down** | open the mod menu |
| **L + R + D-pad Up** | open the game's typed-command prompt |
| Back arrow (bottom left) | up one folder, or close the menu |

## Features

**28 pages, 341 buttons, 78 of the game's ~87 console commands.**

**Items** (116 give-item buttons)
- Unlock all recipes, reforge all items, infinite buckets, vanity masks, hallowed items
- Blocks and materials, gear, potions, and a few useful tools
- **Developer sets** that exist in the game: Red, Cenx, Crowno, Will, Jim, Aaron, D-Town and Fox (wings, helmets, armor, dresses, and the joke item
  *Crowno Devours His Lunch*)
- **Mobile-exclusive items** that were never on PC: Dragon, Titan and Spectral armor, Tizona, Tonbogiri, Sharanga, Vulcan Repeater,
  Ocram's drops, the Fabulous and George's set with Sparkly Wings, and event items (Easter, Valentine's, Oktoberfest, holiday, music boxes)
- Item buttons call the game's own item-spawn function directly, so they're fast on an Old 3DS

**Player**
- God mode, max life 100-500, max mana 100-400, 14 buffs, the game's boost presets, all pets, respawn
- Teleport presets and a dig-to-the-Underworld button

**World**
- Dawn, noon, dusk, midnight and full moon; time speed and world rate
- Rain on/off and strength, fireflies, reveal the whole map, enable Hardmode, flood the dungeon
- Events: solar eclipse, blood moon, frost moon, pumpkin moon, invasion, meteor
- World-generation helpers (a few marked *untested*)

**NPCs and bosses**
- A boss page with the game's boss list, including **Ocram**, the mobile-exclusive boss, and **Lepus**, the Easter boss
- Spawn enemies, spawn town NPCs, spawn all bosses/slimes/worms, clear enemies/NPCs/items, enemy spawns on/off

**Build tools**
- The game's built-in cursor tools: mine, erase, remove, water, lava, caves, cactus, mushrooms, shroom patch, and tile placement

**Random and chaos**
- Random item, buff, enemy, boss, teleport, and weather/time/event
- **Chaos mode**: a random action every few seconds (safe, or with bosses)

**Everything else**
- Display and debug options, Angler quest tools, and **Type a command...** for any other console command
- ON/OFF toggles show their state, and chaos mode shows which setting is selected

## Install

You need your **own dump** of the game. This repo contains no game files and never should.

- **Luma3DS:** put the patch at `/luma/titles/000400000016A900/code.ips` and turn on **Enable game patching** in the Luma config
  (hold **Select** while booting).
- **Azahar:** put it at `<user folder>/load/mods/000400000016A900/exefs/code.ips`.

Built for the **US release, update v5.6.0 (in-game v1.0.5)** only. Other regions and versions will not work and should not be patched.

## Status

Works in **Azahar** and on an **Old 3DS** (Luma3DS).

Known issues:
- Toggle labels track your button presses and can't read the game's real state, so they can drift if you change something another way.
- A few world-generation buttons are marked *(untested)*.
- Each folder takes a moment the first time you open it.
- Back up your saves before trying any mod on real hardware.

## Planned

A drop randomizer (shuffled block and loot drops), quality-of-life fixes, and more.

## Credits

Ghidra, ctrtool, GodMode9, Luma3DS, Azahar, Keystone, Capstone and zig. Reverse engineering and code by **Claude Sonnet 5.5**;
dumping, direction and testing by Phoze. Terraria belongs to Re-Logic. This project is unaffiliated and ships no game content.

As mentioned above this project has future plans and its definitely going to be a big one as im personally obsessed with terraria and also the 3ds. Enjoy!
