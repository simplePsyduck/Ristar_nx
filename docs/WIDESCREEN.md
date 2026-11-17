# Widescreen (16:9 / 16:10) for Ristar

How the wide view works, what was found in the ROM to make it work, and how to check it. Everything was found by
running the ROMs headless (`tools/mdt.py`, `tools/re/checks.py`) and reading the code. **Addresses are the USA /
Europe release's** (CRC32 `6511AA61`); the Japanese release (CRC32 `CE464F0E`) has the same code a little further up
or down, and the same RAM. The code is `src/game/ristar.cpp`.

## Which scenes are wide

| Scene | Mode / step | Wide |
|---|---|---|
| Logos, title, menus, options | `$50` | no (4:3, pillarboxed) |
| A stage: round card, black, fade-in, play, goal and fade-out, bonus rooms | `$0C`, steps 2 and up | yes |
| Between stages (the planet-clear flight, "FLORA CLEAR!!") | `$54` | yes |

`$FFEA00` is the main mode (table at `$3B24`), `$FFEA02` the stage mode's step (table at `$793C`): 2 the round card,
3 black, 4 fade-in, 5 play (pause included), 6 fade-out, `$E`/`$F` into a bonus room, `$10` a bonus room.

## What the game does

| What | Where |
|---|---|
| Foreground layer | `$FFF000`: +0 max camera X, +4 min camera X, +8 / +$C the same for Y, +$10 layout, +$20 camera X, +$24 camera Y |
| Background layer | `$FFF080`, same shape |
| Per-line scroll | built every frame by `sub_014574` (called from the stage loop at `$84E6`) into `$FFF600`: per line plane A, plane B |
| Map | 256-px chunks, 64 a row in the layout (0 = nothing), 16-px blocks of 2 x 2 cells; planes 64 x 32 cells |
| Objects | `$40` bytes; lists headed at `$FFDFF0..$FFDFF8`; +0 routine (table at `$3B84`), +2 flags (bit 0 out of the draw window, bit 3 placed on the screen, not in the stage), +3 animation frame, +$20 / +$24 position, +$28 / +$2A screen position (+128), +$3A art slot or parent |
| Sprite builder | `$4FCA`: per object `$5050`, pieces stored at `$50C8` / `$5130` (`move.w d0,(a2)+`), RAM copy of the table at `$FFEC00` (80 sprites) |

## The view

### The planes

The game keeps each 512-px plane filled with the map from its camera's left edge on: `$7148` draws a column when the
camera crosses a 16-px step, `$76A6` the whole plane. The wide view moves that window **W = 112 px left** (camera
X - 112 .. + 400): hooks at `$7166`, `$7178` (columns), `$71A2`, `$71AE` (rows), `$76AA` (whole plane) take W off
the X the game draws at. 112 because the view reaches from camera - 80 to camera + 400 (the lean below), and the
column the game streams in ahead overwrites the ring 32 px early behind it. A map block an object changes (`$6DEE`) is
written to the plane in the same window (`$6DF4`, `$6DFA`). Left of the background map's column 0, the plane keeps
what the game would have had there, the map 512 px further on (`$71E0`, `$72CC`, `$7322`).

Switched on in the middle of a stage (planes drawn for 4:3), the game's whole-plane routine (`$7686`) is run once
from the top of the stage loop (`$8268`). A state saved by version 0.2 (window from camera - 80) is not redrawn: the
missing 32 px fill in as the camera moves. A redraw is not harmless in every room (Sonata's arena is not what its map
draws: the curtain is).

### The lean (VDP::view_shift)

The game's camera is never touched. The view leans up to E (40 in 16:9) left or right of it:

* **Stages**: the view follows the camera and, near either end of the game's camera limits (`$FFF004` / `$FFF000`),
  leans so that it shows no more than the 4:3 camera can reach: at the start of a stage the picture shows map X
  0..400 until the camera has moved 40 px; at the end it stops where the 4:3 picture stops. The fade-in and the
  goal's fade-out lean the same way (before, the picture jumped 40 px there and showed unstreamed map: states 3, 10).
* **Rooms**: a camera range that fits in the wide view (boss arenas, locked screens: range <= 2E) is shown whole and
  still, centred on the range. If plane A shows nothing on one side of that while it has something just past the
  other, the room eases (2 px a frame) up to E towards its content: Sonata's curtain, drawn from the locked camera's
  left edge on (state 15).
* When the game moves its limits (locking a room, opening it), the limits the view follows ease towards them at 4 px a
  frame, so the view slides instead of jumping; they jump when a stage or a bonus room starts.
* The lean is worked out from the camera the shown scroll table was built for (the table reaches the chip a frame
  after it is built). Not from the table itself: Undertow's second round ripples plane A
  line by line, and the view wobbled with it (state 10).
* The round card and the scenes between stages are not leaned.

### The background follows the lean with the game's parallax

While the game builds the per-line scroll (`$84E6` .. `$84EC`), its camera X holds camera + lean: every band of the
background is placed as the game would place it for a camera that far on, then the camera and plane A's words are
put back (the renderer leans plane A itself). Without this the background slides on its own whenever the view leans
(the start and end of every stage, rooms). Plane A's 224 words are all moved by - lean before the builder and by
+ lean after it, so the words the builder rewrites and those it leaves alone come out the same (telling them apart
by value failed against a room's edge, where camera + lean stands still: plane A slid in the bonus rooms, 0.4).

## Objects and sprites

| Address | What | Wide |
|---|---|---|
| `$8806` (patch) | objects are drawn while 0 <= X - (camera - 128) <= `$1F0` | `$1F0 + 2E` |
| `$50C2`, `$512A` (patch) | the builder keeps 9 bits of sprite X | 10 (`VDP::sprite_x10`) |
| `$50C8`, `$5130` (hook) | each piece | marks HUD / strip / on-screen pieces; leaves out pieces the wide view cannot show |
| `$58D2` / `$58D8` (patch) | "is it on screen" (`$58CE`, screen x `$60..$1E0`), asked by about 60 objects | `- E` / `+ E` |
| `$55F4`, `$57C8`, `$5814`, `$58AE` (hooks) | the object list makes / gives up an entry within its reach (`$12`) of the middle of the 4:3 screen | reaches over 160 (made out of sight) grow by E + abs(lean); smaller ones are triggers in the picture and stay |
| `$42494` (hook) | an enemy asks for its animation frame in an art slot it shares with others of its kind | not asked by one outside the draw window (it took the slot from one on the screen: Flora's bird on the flower, state 2) |

**The sprite budget.** The console shows 80 sprites. The game's draw window is 48 px past the 4:3 edges; the wide
one, with its lean, more. Pieces the wide view cannot show are left out (the piece's 6 bytes written so far are taken
back and its index reused), so they do not use up the 80: Flora's light shafts (16 pieces each, state 4) took 80 and lost
the top-left shaft's lower part.

**Surface strips.** The rising lava (routine `$188`) and the water's foam (`$1D0`, `$54C`) are rows of 32-px pieces
64 px apart, alternating by 32 px every frame, pinned to the camera (camera + `$10` / `$B0`) to cover the 4:3 width.
The renderer draws each of their pieces once more, 320 px over (`VDP::sprite_copy`), so they cover the wide view the
same way. Before, the edges blinked (Scorch's rising lava: report state 13; Undertow's surface: state 8).

**Placed on the screen.** Objects with flags bit 3 (the bonus rooms' captions, the countdown, the goal texts, the
petals) are placed in 4:3 screen coordinates: the renderer keeps them with the screen (offset by the lean), as the
HUD, so they stay centred.

**The HUD** (score `$BC`, lives `$C0`, health stars `$C4`) moves out to the screen edges ("Re-anchor HUD").

### Made or given up at the 4:3 edges

| Where | What | Wide |
|---|---|---|
| `$45464` (`$3F4`) | Flora's petals, made at screen X 320 - 0..99 | + E |
| `$45398` / `$453B6` (`$224`) | Flora's far trees and flowers (3/4 camera speed): hidden while X + 16 < camera, wrapped (+1024) at camera - 64 | from the wide view's left edge |
| `$46604`, `$4D186` (`$33C`, `$06C`) | the items the current carries (Undertow, state 11), made at camera + 320 | + E + lean |
| `$46638`, `$4D068` | ... given up at X <= camera | at the wide view's left edge |
| `$540F0` / `$540F8` | an object that wakes when X - camera is in 0..`$140` | the wide screen |
| `$329D4` / `$329E4` | Sonata's boss, dying, bounces between screen X `$20` and `$120` | between `$20` and `$120 + 2E` of the wide screen |

### The round card

Plane A holds the card over the stage's preview (plane B), drawn for 320 px; past it plane A has leftovers. On each
line and side, where the card reaches the 4:3 edge (its bands, its border, the end of its title bar) its last 16 px
are repeated out to the screen's edge; elsewhere plane A is left out and the preview fills the screen
(`VDP::line_hook`).

### Special rooms and scenes (0.4)

* **Between stages** (mode `$54`): plane A holds half a planet cut by the 4:3 left edge as the flight starts and
  half of the next at the right edge as it ends, with leftovers beside each. `line_hook` keeps plane A's 4:3 part
  only; the view leans +E while plane A touches only the left 4:3 edge, -E while only the right, 0 otherwise (eased
  2 px a frame, jumps while plane A is empty). Sprites lean with it.
* **Sonata's stage after the boss** (stage `$0B`, `$FFE57F`, room locked): the view stays centred; the curtain
  (a 32 px pattern) is carried on to the left on the lines it has reached (checked on two periods first, so the
  arena behind a half-closed curtain is left alone).
* **Shafts no wider than 4:3** (camera range 0, e.g. Riho's boss, stage 8): plane A's floor lines - those with
  pixels in the shaft's middle 160 px - are repeated into the sides. The walls (plane B, with plane A vines along
  the edges) are left alone.
* **Sprite marks** (HUD, foam strips, screen-placed pieces) are set aside when the builder ends (`$503A`) and take
  effect at the sprite table's DMA (`$433A`, `$D44C`): until then the chip still shows the old table.

## The emulator core: the last visible line (0.4)

On line 223 the line counter steps (and may raise a horizontal interrupt) before the vertical interrupt, which comes
788 master clocks into the line. The core raised them the other way round. Ristar's water arms the routine that
loads the underwater colours from its vertical interrupt; a horizontal interrupt left over from line 223 then ran it
at once, and the whole picture above the water was in underwater colours for a frame (state 8, in 4:3 too).

## The Japanese release

Same program, assembled with small differences (no start-of-loop sound test, one sprite loop without the 80-sprite
test, the stage loop's calls in another order, and it refuses to start on a non-Japanese console; the core takes the
console's region from the cartridge header). `tools/re/ristar_sites.py` finds each USA address used in `ristar.cpp`
in the Japanese ROM by the bytes around it (which must occur once), checks the words at the place are equal, and
writes the table (`kJpSites`). The places that differ are given by hand, with the reason, in the tool.

```
python tools/re/ristar_sites.py "<USA rom>" "<Japanese rom>" --write     # after adding an address to ristar.cpp
```

## What was checked (on a PC, headless)

* The 20 save states of the 0.2 report (`ristar_slot1..20.state`), each in 4:3 and 16:9 side by side: the fixes
  below, plus the middle 320 px of the wide picture against 4:3 (lockstep), objects appearing or vanishing inside the
  view, sprite blinking per pixel, visible enemies drawn with another one's art, frame-to-frame movement of each plane.
* All 21 stages of the USA ROM started fresh in 16:9 and moved through; 4 stages of the Japanese ROM.
* `tests/ristar_smoke.cpp` on both ROMs: identification, 4:3 byte-identical, 9 patched words, boot with sound, wide
  frames, save-state replay.

## Known limits

* **Not run on a Switch yet** by us: please send save states of anything that looks wrong.
* **Locked rooms with nothing beside them**: the room is centred, with the backdrop colour on both sides where the
  map has nothing, except for the floor of a 4:3-wide shaft and Sonata's curtain (above).
* **Enemies wake earlier**: the object list's reach grows with the view (40 px more, or the lean on that side).
* **More things are alive** in the wide view, which draws on the game's random numbers: random patterns (debris,
  petals) differ from 4:3. Gameplay is the game's own.
* Some screen-space animations (the bonus countdown) slide in from the 4:3 edge and are seen sliding further.
* About 130 places in the ROM read the camera X directly; the ones above were found from reports and patterns, not
  all were gone through one by one.

## Looking yourself

```
g++ -shared -fPIC -O2 -std=c++17 -pthread -Isrc src/tools/mdtool.cpp src/core/*.cpp src/game/common.cpp \
    src/game/ristar.cpp src/app/palette.cpp -o build/tools/libmdtool.so
python tools/re/checks.py strip  ristar_slot5.state out.png 12          # 4:3 beside 16:9, 12 frames
python tools/re/checks.py centre ristar_slot5.state 120 '0:20,RIGHT:100'
python tools/re/checks.py pops   ristar_slot5.state 300 'RIGHT:150,LEFT:150'
python tools/m68k_disasm.py "<rom>" 0x7148 40       # the scroll routine
python tools/m68k_disasm.py "<rom>" xref 0xFFF020   # who reads the camera
```
