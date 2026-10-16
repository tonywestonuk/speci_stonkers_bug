# "Don't go near the bridge": the Stonkers crash bug, found and fixed

*Stonkers* (Imagine Software, 1983) was a real-time wargame for the 48K ZX Spectrum.
John Gibson wrote the code, the design was by D.H. Lawson and John Gibson, and Paul
Lindale did the graphics. It was clever and ahead of its time, and it was famous for one
thing above all: **it crashed when you tried to cross the bridge.** The crash was
never fixed, and "don't go near the bridge" became a catchphrase.

Forty-three years later, this repository explains what was actually going wrong,
fixes it with 13 bytes, and shows both on video.

| The original game, crossing the bridge | The patched game, same moment |
|---|---|
| ![Original: the Spectrum resets mid-bridge](images/bridge_crash.gif) | ![Fixed: the unit disbands and the battle goes on](images/bridge_fixed.gif) |
| The division runs out of supplies mid-bridge... and the Spectrum resets | "UNIT DISBANDED (NO SUPPLIES)", and the battle carries on |

> The investigation and this write-up were done by Claude (Anthropic's AI model), working
> with Tony Weston, in October 2026. "I" below is Claude.

---

## TL;DR

* When one of your divisions **runs out of supplies on the same tick that it moves**, the game
  jumps into its "UNIT DISBANDED (NO SUPPLIES)" routine with an extra value still on the
  Z80 stack. That routine finishes with a `RET`, which "returns" to the address of the unit's
  own data. **The CPU then runs the division's record (supplies, position, and the route
  you steered) as if it were program code.**
* What happens next depends on those bytes. In my tests about **6 times in 10 the machine
  crashes or resets**. The rest of the time it limps on by luck, possibly with memory quietly
  corrupted.
* It bites late in a battle, when your army has crossed the only bridge and is pushing
  towards the enemy HQ and port, a long way from its supply trucks. Hence the bridge.
* **The fix is one missing `POP`.** Room for it is made by turning a `JP` into a `JR`.
  See [`fix/`](fix/).

## The legend

From John Gibson's recollections (interviews reproduced by
[Spectrum Computing](https://spectrumcomputing.co.uk/entry/4913/ZX-Spectrum/Stonkers) and
[*The Making of Stonkers*](http://www.wizwords.net/the-making-of-stonkers), originally
*Retro Gamer* issue 123):

> "I wasn't given any time to Q&A test. No sooner had I written the last line of code and
> transferred the game to tape, it was whisked off for duplication. It was actually amazing
> that the game only had the one major bug. Maybe it's a testament to my programming
> techniques..."

> "My late dear friend Steve Cain used to play Stonkers a lot and he'd worked out how to
> approach the bridge in such a way as to avoid the crash."

The public were told it was a faulty batch of tapes and offered replacements, but the
replacements were the same version. Gibson wanted to fix the bug, but Dave Lawson wanted him
on *Bandersnatch* instead.

The clue that cracked it came from **Martin Linklater**, who worked with John. He commented on the
thread where Tony was posting progress:

> "The crash was always at the end of the battle."

Until then I'd spent a long time finding nothing wrong with the bridge itself (more on that
below). That one sentence pointed me at the right place.

---

## How a Stonkers division works

![The battle map](images/battle_map.png) ![Zoomed in on the bridge](images/bridge_zoomed.png)

The battlefield is a 160×96 grid of 8×8-pixel tiles. Positions are in pixels: x runs
0 to 1279 and y runs 0 to 767. You give orders in the zoomed view: put the X cursor on a
division, press fire, steer the cursor to where you want it to go, and press fire again.
There's no pathfinding. **The game records your cursor movements step by step and the unit
replays them.**

Each of your 16 units has a 97-byte record (`$E4D0` + 97×n):

| Offset | Contents |
|---|---|
| `+$00` | flags (bit 0 lost, bit 1 in combat, bit 5 supply truck, bit 7 moving) |
| `+$01` | unit number (low nibble) and combat partner (high nibble) |
| `+$02` | tick counter; every 8th tick costs one point of supplies |
| `+$03` | supply strength (BCD) |
| `+$04` | combat strength (BCD) |
| `+$05`/`+$06` | terrain class and speed |
| `+$07` | steps of route left |
| `+$08`–`+$47` | **the route: up to 128 steps, one 4-bit direction per step** |
| `+$48`–`+$51` | position within its map cell, the cell number, and its 8 neighbouring cells |
| `+$52`–`+$55` | x, y |
| `+$56`– | size, sprite pointers |

## The bug

The routine at `$C87D` updates one player unit per pass of the main loop. For a moving unit
it takes this path (simplified from the disassembly):

```
C8F8  PUSH HL          ; save the unit's address ...
 ...                   ;   ... decode up to <speed> route steps ...
C96D  CALL $D4D9       ;   move, and check the terrain (water = "UNIT LOST UNDER WATER!")
C973  POP HL           ; ... and restore it (balanced so far)
      JR NC,$C995
 ...
C995  PUSH HL          ; save the unit's address AGAIN       stack: [unit] [return]
      BIT 5,(HL)       ; a combat division, not a supply truck?
      JR Z,$C9E0
 ...
C9E0  ...              ; every 8th tick of movement:
C9F1  LD A,(HL)        ;     supplies := supplies - 1  (BCD)
      SUB $01
      DAA
      LD (HL),A
      JR NZ,$CA01      ;     some left: carry on (the address is popped later, at $CA67)
C9F8  DEC HL           ; none left: "UNIT DISBANDED <NO SUPPLIES>"
      DEC HL
      DEC HL
      LD DE,$5780
      JP $CB40         ; -> remove the unit, queue the message, update the side panel,
                       ;    and finally  ... JP $A246 -> ... RET
```

The code at `$C9F8` is a **shared tail**. It's also where a division that's standing still
goes when its supplies run out:

```
C8B3  INC HL           ; (standing-still path: nothing has been pushed)
      LD A,(HL)
      SUB $01
      DAA
      LD (HL),A
      JP Z,$C9F8       ; out of supplies -> the same tail
```

From the standing-still path the stack holds just the return address, so the final `RET`
goes back to the main loop. From the moving path there's an extra word on top: the unit's
own address, pushed at `$C995`. So:

```
standing still:   [return to main loop]            RET -> main loop       ✓
moving:           [unit address][return]           RET -> the unit's data ✗
```

The CPU jumps to `$E4D0` (or wherever that unit's record is) and starts executing it.

### Why your route matters

The record is mostly route, and each step is stored as a 4-bit direction. Two steps in the
same direction make one byte, and every byte is also a Z80 instruction:

| Steered | Nibble | Two steps make | which the Z80 runs as |
|---|---|---|---|
| NW | `0` | `00` | `NOP` |
| NE | `1` | `11` | `LD DE,nnnn` |
| N  | `3` | `33` | `INC SP` (the stack is now out of line) |
| SW | `4` | `44` | `LD B,H` (harmless) |
| SE | `5` | `55` | `LD D,L` (harmless) |
| S  | `7` | `77` | `LD (HL),A` (writes to memory) |
| W  | `C` | `CC` | `CALL Z,$CCCC` (calls into the middle of the program) |
| E  | `D` | `DD` | `IX` prefix |

The bytes that follow matter too. The routine that replays a route rotates the buffer as it
consumes it (with `RRD`), leaving a residue mixed with whatever was in the A register. After
that come the unit's map-cell numbers and its position.

In the [crash video](videos/bridge_crash_ORIGINAL.mp4) the route was pure south-west, a
harmless run of `LD B,H`. But the residue a few bytes further on read `C4 4D 44`, which is
`CALL NZ,$444D`, straight into screen memory, and the Spectrum reset. Had it got past that,
the next bytes are the unit's position within its map cell, the cell number, and its eight
neighbouring cells. For a unit standing on the bridge cell (`$86`) they read
`3B 86 72 9A 85 71 99 87 73 9B`: a `DEC SP`, and three instructions that write to memory
(`LD (HL),D`, `LD (HL),C`, `LD (HL),E`).

So whether the game survives really does depend on **where the unit is and how you
steered it there**. I can't know what Steve Cain actually did, but that fits his trick of
approaching the bridge in a particular way.

### Why "the bridge"?

The game's computer side waits on the far bank, and to win you must occupy its port and HQ
(or destroy every enemy division). There's one river and **one bridge**, so every attacking
move funnels across it and on into enemy territory, far from your supply trucks. That's when
divisions start running dry while on the march: *at the end of the battle*, as Martin
remembered. To a player, the game "always crashed after I went over the bridge".

That last link is inference: nobody can replay a 1984 living room. The mechanism isn't. I
reproduced it in full simulated battles where the army was marched across the bridge
towards the enemy HQ, and the crashes came from exactly this path.

## The fix

The moving path needs to throw away its stray stack entry before joining the shared tail,
and that needs one more byte. Shortening `JP $CA3E` to `JR $CA3E` frees one, so the
following instructions shuffle up a byte and a `POP BC` takes the gap:

```
        original                         patched
C9EB    JR Z,$C9F1                       JR Z,$C9F0
C9ED    INC HL                           INC HL
C9EE    JP $CA3E                         JR $CA3E          ; 3 bytes -> 2
C9F1    LD A,(HL)                 C9F0   LD A,(HL)
        SUB $01                          SUB $01
        DAA                              DAA
        LD (HL),A                        LD (HL),A
        JR NZ,$CA01                      JR NZ,$CA01
                                  C9F7   POP BC            ; <- the missing instruction
C9F8    DEC HL ...                C9F8   DEC HL ...        ; shared tail untouched
```

Nothing else jumps into those bytes (I checked every `JP`/`JR`/`CALL`/`DJNZ` in the program).
The standing-still path still lands on `$C9F8` exactly as before.

* **Patch a tape:** `python3 fix/make_fixed_tap.py STONKERS.TAP STONKERS_FIXED.TAP`. It checks
  the bytes it expects, patches them and recalculates the block checksum.
* **POKEs** for an emulator (apply while paused), also in [`fix/POKES.txt`](fix/POKES.txt):
  ```
  POKE 51692,3: POKE 51694,24: POKE 51695,78: POKE 51696,126: POKE 51697,214: POKE 51698,1: POKE 51699,39: POKE 51700,119: POKE 51701,32: POKE 51702,10: POKE 51703,193
  ```

The game isn't included in this repository. It's available from
[Spectrum Computing](https://spectrumcomputing.co.uk/entry/4913/ZX-Spectrum/Stonkers).
The tape I tested has MD5 `f0c54d9f56300e177f47c62618493c2d`.

## How sure am I?

All testing was in an emulated 48K Spectrum (SkoolKit's Z80 simulator, driven by the scripts
in [`tools/`](tools/)), playing the game through its real controls.

| Test | Original | Patched |
|---|---|---|
| Divisions running out of supplies while moving near the bridge (100 runs) | **61 crashed**, 27 survived by luck | 0 crashed, 0 stray executions |
| Full automated battles: the whole army marched across the bridge to the enemy HQ and port (20 battles, 30 game-minutes each) | **2 crashed**, both via this bug; 8 reached an end screen; 10 were still being fought | 0 crashed; 10 reached an end screen (including both that crashed before); 10 were still being fought |
| Bridge crossings with healthy supplies, at many angles, with and without combat | no faults | no faults |
| All four endings (victory ×2, defeat ×2) and starting the next battle | work | work |

The supply test is the dependable way to reproduce the bug (`tools/fuzz4.py`), and gives the
same numbers on every run. Full battles are chaotic: whether one crashes depends on exactly
when and where divisions run dry, and even on how the emulator is stepped. With the harness
as committed (frame by frame), a 30-battle run happened not to produce a crash at all.

Not tested: the patched tape on real hardware or in mainstream emulators such as Fuse.

## Videos

* [`videos/bridge_crash_ORIGINAL.mp4`](videos/bridge_crash_ORIGINAL.mp4): **the bug.** An armoured
  division with nearly empty supplies is ordered across the bridge through the normal controls.
  Mid-crossing its supplies hit zero and the Spectrum resets to "© 1982 Sinclair Research Ltd".
  (To make it happen on camera the division was placed by the bridge with its supplies set to 2.
  Everything else is the unmodified game.)
* [`videos/bridge_crossing_FIXED.mp4`](videos/bridge_crossing_FIXED.mp4): the same scene on the
  patched game: "UNIT DISBANDED (NO SUPPLIES)", and the battle carries on.
* [`videos/stonkers_gameplay.mp4`](videos/stonkers_gameplay.mp4): the game working normally: loading,
  menus, the battle map, zooming in on the bridge, sending a division across.
* [`videos/full_battle_autoplay.mp4`](videos/full_battle_autoplay.mp4): ten minutes of a whole battle
  played by the test robot (this one didn't hit the bug).

![Crash sequence](images/crash_sequence.png)
![Fixed sequence](images/fixed_sequence.png)

## A second bug that never bit

When a supply ship docks, `$DE0D` looks for a free slot to bring a new supply truck ashore.
It starts with `PUSH BC`, and if all four trucks are still alive it `RET`s without popping.
So it too "returns" into data: the table of starting positions at `$E450`.

By pure luck those bytes, `E8 04 00 00 C0`, decode as `RET PE` (not taken), `INC B`, `NOP`,
`NOP`, `RET NZ` (taken), which returns to exactly the right place. The enemy's version lands
on `$E490` and also happens to reach a harmless `RET NZ`. It runs every time a ship docks
with a full set of trucks, and has never hurt anyone. I found it by accident when my own
test harness edited the start positions and the game promptly fell over. I've documented it
but left it alone.

## The code, and the man who wrote it

I spent a long time inside John Gibson's code for this, and I came away with a great deal of
respect for it.

Stonkers is a real-time strategy game with two map scales, a computer opponent, terrain that
changes how fast you move, supply lines, ships running on schedules, combat, a scrolling news
ticker and a battle clock, all in about 40K of a 48K machine. Gibson told *Retro Gamer* that he
wrote it on one of Imagine's Sage IV computers, never directly on a Spectrum, as the last of
three games he wrote in 1983, against a pre-Christmas deadline. You can feel that economy on
every page:

* **Nothing is wasted.** Most of the top-left corner of the terrain map is open sea, where
  no unit can ever go, so the game keeps its working variables there, and even runs its
  start-up code there before wiping it. The 32 ticker messages are stored as plain text *in
  screen memory*, in the rows between the lines of the unit-statistics panel, permanently
  coloured black on black. The end-of-battle text sits in the printer buffer. The computer's
  direction preferences are tucked into unused slots in the tile graphics.
* **The terrain is beautifully cheap.** Each third of the map has its own 256-tile set, and
  the tiles are numbered so that comparing a tile number against three thresholds tells you
  whether a square is passable and how fast you can cross it.
* **The computer opponent is more thoughtful than it lets on.** It follows a road network of
  map cells (the bridge is written into it, with a coin toss at the junction beyond it), keeps
  influence maps where every unit spreads weight into its neighbouring cells, and in the
  difficult game hides its units unless one of yours is close by. According to the
  *Retro Gamer* interview, the AI's approach was borrowed from *Eastern Front*.
* **It's written by someone who thought in registers.** The difficulty setting is literally
  the offset of one `JR`. The sprite and cursor routines jump into the middle of their own
  shift sequences, and combat juggles both fighting units at once with `EXX` and `EX AF,AF'`.

The bug itself is the most understandable kind. Sharing the tail end of a routine between two
callers is exactly what you do when every byte counts. Here the two callers arrived with
different things on the stack, and the slip only shows when a division's supplies run out on
the precise tick it's moving, which you'll rarely see until late in a long game. Without a
debugger, and with no time at all for testing, that is a very easy one to miss. (The second,
harmless bug is the same kind of slip, and survived on the luck of where some numbers
happened to sit. It feels a bit like a gentle joke from the past.)

Having read the code, I think his own verdict holds: *"It was actually amazing that the game
only had the one major bug. Maybe it's a testament to my programming techniques..."* I'd say
it was. Hundreds of simulated crossings, skirmishes and battles turned up nothing else that
could crash the game. The bridge bug became a running joke, but the game behind it is a fine
piece of work.

*In memory of John Gibson, and of Steve Cain, who knew how to cross the bridge.*

## Repository layout

```
fix/            make_fixed_tap.py (patcher) and POKES.txt
tools/          the emulator harness and every script used in the investigation (see tools/README.md)
disassembly/    stonkers.ctl - a SkoolKit control file naming the routines (no game bytes)
videos/         the four videos above
images/         stills and GIFs used in this README
```

To rebuild a commented disassembly from your own copy of the game:

```sh
cd tools && ./setup.sh /path/to/STONKERS.TAP
./venv/bin/sna2skool.py -H -c ../disassembly/stonkers.ctl work/stonk.z80 > stonkers.skool
```

## Thanks

To Martin Linklater for the clue that cracked it. To Tony Weston for asking, and for not taking
"the bridge code looks fine" for an answer. To Richard Dymond for SkoolKit, which made the
emulator, the disassembler and the tracing possible. And to Spectrum Computing, and the
preservation community generally, for keeping these games, and the stories behind them,
available.
