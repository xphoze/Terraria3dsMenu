# How the Terraria 3DS developer tools were found and unlocked

A technical write-up of how a debug console and a developer settings screen that Re-Logic / 505 Games left inside the
retail Nintendo 3DS build of Terraria were located, unlocked, and turned into a clickable in-game mod menu.

Everything here was done on a dump of a legally owned copy. **No game files are included or needed in this repository**;
only small patch files and the scripts that build them. All addresses refer to the decompressed `code.bin` of the
**update title v5.6.0 (title ID `0004000E0016A900`, game version shown in-game as v1.0.5)**, loaded at `0x100000`
(file offset = address - `0x100000`). The base title is `000400000016A900`.

> **Status:** developed in the **Azahar** emulator and then confirmed working on an **Old 3DS** running Luma3DS
> (section 8 covers the one hardware-only bug that had to be fixed). Only the USA update v5.6.0 build has been tested.

---

## 1. Getting the code out of the game

1. Used **GodMode9** on the console to build standard CIAs of the installed base game and its update.
2. Extracted both with **ctrtool** (`--exefsdir`, `--romfsdir`, `--decompresscode`). The CIAs GodMode9 builds are already
   decrypted, so no keys were needed.
3. `exefs/code.bin` is stored compressed in the title; ctrtool's `--decompresscode` gives the real 5,480,448-byte image.
4. The ExHeader gives the memory layout, which is needed to load the binary correctly:

| segment | address | size |
|---|---|---|
| text | `0x100000` | `0x4C0BBC` (page-rounded to `0x4C1000`) |
| ro | `0x5C1000` | `0x672DC` |
| data | `0x629000` | `0x109A8` |
| bss | after data | `0x2A1C50` |

The three file segments add up to exactly `0x53A000` = the decompressed `code.bin` size, a good sanity check.
On an Old 3DS the game gets 64 MB of application memory (124 MB on New 3DS).

## 2. Static analysis setup

* Loaded `code.bin` into **Ghidra** as a raw binary, processor `ARM:LE:32:v6`, base address `0x100000`.
* Auto-analysis found about 8,500 functions. All symbols are stripped, so every name in these notes is ours.
* The binary mixes **ARM and Thumb** code. This matters later: the game calls some functions with `blx` (Thumb) and others
  with `bl` (ARM). Calling a Thumb function with `bl` from injected ARM code crashes immediately with
  "Undefined Instruction" at the function's first bytes.
* Ghidra's decompiler hides some arguments (a register that is never "used" in the callee's body but is live on entry).
  Reading the raw disassembly was necessary several times.

## 3. Finding the developer features: strings first

Dumping every printable string in `code.bin` (only ~6,100) immediately showed things that should not be in a retail
game:

* A list of console command names: `godmode`, `item `, `spawnhouse`, `boss `, `tp `, `hellavator`, `allrecipes` ...
  about 87 in total, each with the on-screen message it prints ("Godmode enabled", "Wooden Door created" ...).
* A `help` text and the class name `CheatKeyboard`.
* Textures named `console_background`, `console_add`, `console_substract`, `console_toggle` (present in the romfs).
* `15DevSettingsMenu` (C++ RTTI), `devsettings_bottom`, `devsettingsbutton.xml`, and in the romfs two UI layouts,
  `devsettings.xml` (a scrolling list with a back button) and `devsettingsbutton.xml` (one green list entry).

So there were two separate things to find: a **text console** and a **settings list screen**.

## 4. The text console

### 4.1 Where it lives
Cross-referencing the command strings leads to one huge function, **`FUN_003d339c`** (about 19 KB), which takes the
command text as a game string object and dispatches on it. Two callers:

* `FUN_002460c8` lower-cases whatever text was typed, calls the console, and shows the result in a label.
* `FUN_001c6c54` runs the literal command `boostgod` when a flag is set.

### 4.2 How text gets in
`FUN_002465e0` (a per-frame update method of the game-state manager) has this block (`0x2466e4`):

```
if (manager->flag_0x78) {
    manager->flag_0x78 = 0;
    show "Command: " in the console label;
    open the text-entry keyboard (FUN_001749dc), result goes to FUN_002460c8
}
```

So **writing 1 to the byte at `GameStateManager + 0x78` makes the game open its own "Command:" keyboard**. In Azahar
that appears as the emulated 3DS software-keyboard dialog; on hardware it would be the real software keyboard.
Nothing in the retail game ever sets that flag, which is why the console is unreachable.

### 4.3 Finding the manager object (GDB)
Azahar has a GDB stub (`use_gdbstub=true`, port 24689). A small RSP client (`tools/gdbc.py`) can read memory while the game
runs. Notes: send `?` first on every connection (an unsolicited stop packet arrives), each connection halts the game until
you send `c`, and `Ctrl-C` break is not supported.

Scanning memory for the manager's vtable pointer found the singleton at **`0x8D52AC`**. The input manager was found the
same way at **`0x8D4C18`**.

### 4.4 Reading buttons
The game turns hardware buttons into numbered *actions* stored in two hash sets inside the input object: current frame at
`+0x1E0`, previous frame at `+0x210`. `FUN_005337F0(set, &id)` tests membership. Mapping (from `FUN_003FA790`):

| action | button | action | button |
|---|---|---|---|
| 1 | A | 8 | L |
| 2 | B | 9 | ZL |
| 3 | X | 5 | R |
| 4 | Y | 6 | ZR |
| 0xB | Start | 0xC/0xD/0xE/0xF | D-pad Up/Right/Down/Left |

An "edge" (just pressed) is: in the current set and not in the previous set. The game's own helper `FUN_00524028` also
checks two flag bytes that stay non-zero in-world, so it never fires during gameplay; the patch does the edge test by hand.

### 4.5 The hook
Patching is done with **IPS** (Luma3DS `luma/titles/<id>/code.ips`, and Azahar `load/mods/<id>/exefs/code.ips`). An IPS
can only overwrite bytes in place, so extra code needs somewhere to live. The text segment is page-padded: bytes
`0x5C0BBC-0x5C1000` (1,092 bytes) are zero and executable. That is the "code cave".

* The first instruction of the main per-frame update `FUN_001F7AE4` (`0x1F7AE4`) is replaced by `b 0x5C0BBC`.
* The cave runs the displaced instruction, saves `r0-r3` and `d0`, then tests the combo **hold L + R, press D-pad Up**.
* If it fires: set `manager+0x78 = 1` and run the game's own "open console" block through a small trampoline that
  rebuilds the prologue frame of `FUN_002465e0` and jumps to `0x2466E4`.

An earlier version hooked the manager's update instead; it worked on menus but not in a loaded world (the manager update
does not run in-world), which is why the hook moved to the main update.

### 4.5.1 A command mailbox
To run commands without typing, the cave also checks a mailbox in the free tail of the last bss page:
`0x8DBF00` = flag (1 run command, 2 switch screen, 3 open keyboard), `0x8DBF04` = lower-case ASCII command. It builds a
game string with `FUN_00152090`, calls `FUN_003D339C`, and frees it with `FUN_0014B594`. The menu uses this.

## 5. The dev-settings list screen

### 5.1 The state machine
The game keeps screens as persistent **state objects** in *layers* (`GSM+0xC..0x10`, 16-byte entries
`{key, stackBegin, stackEnd, cap}`). The registry function `FUN_0024434c` creates and registers every screen by name.
`devsettings_bottom` is registered like all the others. Relevant calls:

* `FUN_0012CB80(GSM, &string)` pushes the named state onto its layer.
* `FUN_0012CCCC(GSM, oldState, &string)` replaces one state with another.
* Strings are always passed **by reference** (the address of the string slot); passing the contents crashes.
* A state's first update runs its lazy init through `FUN_00400984`: `if (!inited) { vtable[3](); inited = 1; }`.
  A state whose slot 3 is empty (for example `empty`) crashes there.

Pushing `devsettings_bottom` (L + R + D-pad Down in the menu build) shows the screen. Its original content is seven
entries that Re-Logic used in development:

| label | internal name |
|---|---|
| Floating texts: On/Off | `showtexts` |
| PrecisionMode: L/R hold / swap 2 / rens / toggle | `precisionmode` |
| Copy players/worlds to SDMC | `saveworlds` |
| Load players/worlds from ROM | `loadworlds` |
| Force save world (even tutorial) | `forcesave` |
| Go to main menu | `gotoMainMenu` |
| Go to PIA test | `gotoPiaTest` |

(`Go to main menu` drops you out of the world, handle with care.)

### 5.2 How the list is built
`FUN_00228F94` is the screen's lazy init. It first runs a function-local-static initializer that fills a global table
(a `std::vector` of 24-byte records at `0x8D4BA8`) with those seven entries, then builds one button per record from
`devsettingsbutton.xml`. A record is `{ name string, label vector, label index, toggle flag }`. All buttons call one
shared handler, `FUN_002286A4`.

The list is built **once per screen object**. Building it again *appends* more buttons instead of replacing them.

## 6. Turning it into a real menu

### 6.1 Replace the table at the right moment
Hook `0x228FE0`, right after the game's initializer and right before it reads the table, and swap in our own records
(`menu.c: prep_stub / menu_prepare`).

### 6.2 One screen per page
Since a screen builds its list only once, **each menu folder is its own copy of the dev-settings screen**, created the
first time you visit it with the same three calls the game's registry uses: allocate `0x74` bytes
(`FUN_00145C94(0x74, 0x5D252C)`), run the constructor `FUN_00229A78`, register with `FUN_00109610`. Navigating is a
`replace` to the next page's screen. Nothing is ever rebuilt.

### 6.3 Button presses
The handler `FUN_002286A4` is hooked at its first instruction. The pressed button's registered name comes from
`FUN_00461350(out, widget)`; the `widget` argument is the handler's second parameter (still in `r1`), which Ghidra's
decompile hides. Names are tiny action strings: `m:<page>` navigate, `c:<command>` run through the mailbox, `k:` open the
typed-command keyboard, `r:*` random/chaos, `t<bit>:...` toggles.

### 6.4 Back and close
The screen's native back arrow originally goes to `optionsmenu`. Both back handlers are redirected: on a sub-page they
go up one folder; on the root they remove the screen's layer from the manager's layer list so the HUD underneath takes
over again. (Layers get a new numeric key on every push, so the code finds the layer by the screen object, not by key.)

### 6.5 Labels that change
To show ON/OFF state the menu reuses the builder's own calls to fetch the list's grid widget, take item N, find its
`text` child and set new text (`FUN_00575D64`, `FUN_005344EC`, `FUN_005755DC`, `FUN_00149038`).

The text widget wants a **UTF-16 string**. The builder converts each label with `FUN_001379D8(dest, &narrowString)` (which
widens every byte to two) and releases the result with `FUN_0014F7B8`. The first version skipped the conversion and
showed a huge string of random characters even though the button still worked. The toggle state itself is tracked by the
menu (one bit per toggle), since the game's real state is not readable in a uniform way.

### 6.7 Fast item giving
The console's `item` command spends most of its time matching the typed text against dozens of command names (each step
allocates temporary strings), then looks up the item's name and prints two log lines. That produced a visible hitch on an
Old 3DS. The console ends by calling `FUN_003DBE24(type, count, flagMask, prefix)`, a clean "spawn this item at the player"
helper (`count = 0` gives a full stack, `flagMask = 0` always gives). Item buttons, Random item and chaos mode now call it
directly.

### 6.6 Memory
* Code and data do not fit in one place. The build script (`tools/build_menu.py`) compiles the menu in C
  (freestanding ARM11, via `zig cc`), then places every function and every page's data into **dead code**: functions
  that nothing references. A region only counts as dead if (a) no branch instruction anywhere in the binary targets
  the inside of it from outside, and (b) no data word anywhere points into it. Ghidra's own reachability analysis
  proved unreliable and is not used.
* The finished menu is about 14 KB of code and data (28 pages, about 340 buttons), spread across 15 dead regions. When
  the first batch of regions filled up, a second scan (functions of 200+ bytes with no references into them) turned up 74
  more clean candidates; the build script places each function and each page's packed data first-fit, largest first.
* Working RAM is the only real limit: the last bss page ends at `0x8DC000` and `0x8DD000` is unmapped (using it crashed
  the game). The menu state lives in the 944-byte tail of that page, which is why pages are limited to 16 entries.

## 7. Dead ends and bugs worth knowing about

| symptom | cause |
|---|---|
| Undefined Instruction at the first bytes of a function | called a Thumb function with `bl` instead of `blx` |
| PC = 0 after switching screens | passed a string's contents instead of a reference to it |
| Menu opens but shows the 7 defaults | the game's one-time initializer runs after your table is written; hook after it |
| Folder buttons duplicate and taps "fall through" | rebuilding a screen appends to its grid; use one screen per page |
| Back button only works once | each new layer gets a new key; find the layer by object |
| Crash entering `player_bottom` / `empty` | those states expect different widgets / have no init |
| Replacing the HUD's own bottom state drops you to the main menu | the HUD swaps top and bottom screens together |
| Press handler gets garbage name | the name helper also needs the pressed widget in `r1` |

## 7.1 The hardware-only crash (and how Luma's dump solved it)
The first run on a real 3DS crashed with a data abort and left `/luma/dumps/arm11/crash_dump_*.dmp`. The format (Luma
v1.3): a 0x28-byte header (magic `0xDEADC0DE 0xDEADCAFE`, version, processor, core, type, then sizes of the register,
code, stack and extra sections), then the registers (`r0-r12, sp, lr, pc, cpsr, ...`), the code bytes **ending at the
faulting instruction**, and the stack. Decoding it showed:

* `pc = 0x533804`, the instruction `ldrsb r0, [r0]` inside the game's tiny "is this action active" helper, with `r0 = 0x40`.
* `lr` was inside the menu code, so the patch had loaded and run on the console.
* Cause: the input manager is built lazily. On hardware our per-frame hook ran before it existed, so the table pointer it
  reads still held junk. Azahar always had it ready.
* Fix: only touch the input sets once both table pointers look like heap pointers (`0x08000000-0x10000000`), in both the
  C code and the assembly cave.

The same dump also confirmed the console runs the same code that was analysed (the code bytes matched the dump exactly).

## 8. What the commands do (verified in Azahar)
`life N` / `mana N` set **maximum** life/mana; `buff N` uses PC buff IDs; `item N` uses PC item IDs; `npcspawn N` uses PC
NPC IDs; `tp X Y` uses tile coordinates; `time N` / `rate N` / `spawnrate N` are multipliers; `boss N` is an index into
the game's own list (0 King Slime, 1 Eye of Cthulhu, 2 Skeletron, 4 Wall of Flesh, 5 Lepus, 6 The Destroyer,
7 Skeletron Prime, 8 The Twins, 9 Ocram, 10 Plantera, 12 Brain of Cthulhu, 13 Golem).

## 9. Other things found along the way
* The game's item list reaches ID 22959. IDs 5001-5079 are mobile/event exclusives (Dragon/Titan/Spectral armor,
  Tizona, Tonbogiri, Sharanga, Vulcan Repeater, Ocram drops, the Fabulous/George's set, Easter/Valentine/Oktoberfest
  items, music boxes). The PC developer sets (Red, Cenx, Crowno, Will, Jim, Aaron, D-Town, Fox) are present too.
* The world item array is 200 entries of 0x84 bytes; its spawner `FUN_003C9674` has 73 callers (block drops, enemy loot,
  thrown items, the console). Its item type is loaded at `0x3C9684` (`ldr r6,[sp,#0x40]`), which is the natural hook point
  for a drop randomizer: the caller's `lr` is still intact there, so block drops can be told apart from thrown items.
  This is research only; the shipped patch does not hook it.
* Boss indexes for the `boss N` command (identified by spawning each one): 0 King Slime, 1 Eye of Cthulhu, 2 Skeletron,
  4 Wall of Flesh, 5 Lepus (Easter), 6 The Destroyer, 7 Skeletron Prime, 8 The Twins, 9 Ocram (mobile-only), 10 Plantera,
  12 Brain of Cthulhu, 13 Golem. Indexes 3 and 11 spawn nothing visible; 14 and up do nothing.

## 10. Tools used
GodMode9, ctrtool, Ghidra (headless scripts in `ghidra_scripts/`), Azahar (portable, with its GDB stub), Python with
Keystone and Capstone, zig (as a cross-compiler for ARM), and `pyelftools`.

## 11. Reproducing it
1. Dump your own game with GodMode9 and extract `code.bin` with ctrtool.
2. `python tools/build_menu.py <code.bin> patches/menu.ips`
3. `python tools/build_console_patch2.py <code.bin> patches/console_combo.ips`
4. Merge `door_prompt_off.ips`, `console_combo.ips` and `menu.ips` into one IPS (strip the `PATCH`/`EOF` markers, join
   the records, add them back) and install it as `code.ips`.
5. Open the menu with **L + R + D-pad Down**; the typed console is **L + R + D-pad Up**.

Every build script asserts the original bytes at each patch site, so it refuses to build against a different version.
