# Egg-Farm Platformer — Reimplementation Specification

A platform-neutral specification of a single-screen arcade platform game: the player collects 12 eggs per screen while hens, a
duck and lifts try to stop them. It describes **rules, numbers and algorithms**; any language/toolkit can implement it. Algorithms are given in Python
and are normative (they are transliterations of a model that was compared pass-by-pass against the original program).

**Companion files** (in this directory): **`assets/`** holds the art as **PNG images** that any image viewer can open (tiles, player/duck sprites, hen sprites) plus the plain-text files `assets/layout_0.txt` … `layout_7.txt` (the eight level layouts) and `assets/death_tune.txt`; their format is defined in §5.1. The hen start table and the lift columns are in this document (§10.1, §13). Everything else needed to reproduce the game is in this document.

Contents: 1 Scope · 2 Conventions · 3 Clock · 4 Display · 5 Assets · 6 State · 7 Input · 8 Main loop · 9 Player · 10 Hens · 11 Duck ·
12 Scoring and clocks · 13 Lifts · 14 Game flow · 15 HUD · 16 Audio · 17 Levels · 18 Acceptance tests · 19 Quirks.

Labels: **[V]** verified against the original, instruction by instruction; **[M]** measured on the original; **[R]** read from code, not mechanically verified; **[A]** assumed (value not observable).

---

## 1. Scope and interface

Only the game itself is specified: no title screen, menu, high-score table, key redefinition or instructions.

**Configuration**

| Parameter | Default | Meaning |
|---|---|---|
| `players` | 1 | 1..4 players, alternating, one attempt (life) each turn |
| `keys.up/down/left/right` | Q / A / O / P | climb up, climb down, walk left, walk right |
| `keys.jump` | M | jump (direction = held left/right) |
| `keys.pause` | H | edge-triggered toggle; while paused nothing advances |
| `keys.exit` | ESC | edge-triggered; ends the game at once |

All other values are fixed here (5 lives, 12 eggs, start at level index 0).

**Result**: the final 6-digit score (0..999999) of every player (array of `players` integers), returned when all players have lost all lives or when exit is pressed
(points already earned in the attempt in progress count; no bonus conversion on exit).

---

## 2. Conventions

* Logic coordinates are **y-up pixels** on a 256×192 playfield: `x` 0..255 left→right, `y` 0..191 bottom→top. All arithmetic is on unsigned 8-bit
  values unless stated; `& 255` is shown where wrap-around matters.
* The playfield is a grid of 8×8-pixel **cells**: 32 columns × 24 rows. Screen rows 0-2 (top) are the HUD (§15); the **tile map** covers the lower 21 rows.
* **Tile map**: 672 bytes, 32 columns × 21 rows, row 0 = bottom row. `tile(x, y) = map[(y >> 3) * 32 + (x >> 3)]` (x, y masked to 0..255).
  Reads with index ≥ 672 (only reachable above y = 168) return 0 **[A]**.
  Map row *n* is drawn on screen row `23 − n`; map pixel row `y` is screen line `191 − y`.
* `sgn`-style direction variables use +1 / −1 (stored as 255 in the original; a port may use signed ints).

---

## 3. Clock model (the most important thing to get right) [V]+[M]

The original has no frame synchronisation: game speed is the running time of its main loop on a 3.5 MHz CPU. Speeds are therefore derived quantities. A faithful
implementation emulates the loop in **passes** against a work budget of **3,500,000 work-units (WU) per second**; the acceptance criteria are the wall-clock numbers of §3.3 (±5 %).
A fixed ticks-per-second shortcut is not acceptable (it runs 10–40 % too fast) unless it uses the per-state tick periods of §3.2.

### 3.1 Passes and ticks

* One **pass** is one iteration of the main loop (§8). A counter `pass_cnt` counts passes down; when it reaches 0 it is reloaded with **130** and the **tick body** runs
  (everything "once per tick": walking, hens, duck, timers, pick-ups, lifts). A tick = 129 ordinary passes + 1 tick pass.
* Per pass (not per tick): the airborne physics (vertical countdown), the falling whistle, the pick-up arpeggio, the hen-vs-airborne-player test.

**Measured pass costs (WU)**

| Pass kind | Cost |
|---|---|
| ordinary pass, player on ground/ladder | 157 |
| ordinary pass, player airborne (`air != 0`) | 335 |
| tick pass, standing still | 33,700 |
| tick pass, walked this tick (incl. step beep every 4th tick) | 34,900 (+beep cost) |
| tick pass, climbed this tick | 27,500 (+beep cost) |
| tick pass while airborne (no whistle, no redraw) | 15,000 |
| extra for a pass that performs a vertical step (`land_check` redraw): rising / falling / each drift tick | +10,700 / +26,400 / +15,000 |
| beep(H, L) (blocking) | `L · (26·(H or 256) + 47)` — also for the whistle and the pick-up arpeggio |

```python
budget += dt_seconds * 3_500_000
while budget > 0:
    cost = 335 if air != 0 else 157
    if tick_pass:
        cost = 15_000 if air != 0 else 27_500 if climbed else 34_900 if walked else 33_700
    if land_check_ran_this_pass:
        cost += 10_700 if vdir == +1 else 26_400 if air == 2 else 15_000
    cost += sum(L * (26 * (H or 256) + 47) for (H, L) in beeps_issued_this_pass)
    budget -= cost
    run_pass()
```

### 3.2 Resulting tick periods
standing 15.4 ms (65 ticks/s) · walking 16.2 ms · climbing 14.8 ms · airborne ≈ 22 ms (45 ticks/s) · riding a lift ≈ 19.5 ms.

### 3.3 Per-component cadence (ticks)

| Component | Period |
|---|---|
| walk/climb step, airborne horizontal step | every tick, 1 px |
| hen update (moves ONE hen, round robin over 5 slots) | every 3 ticks (2 when level index ≥ 32) |
| duck update | every 12 ticks |
| lift update | every 2 ticks, 1 px |
| TIME −1 | every 10 ticks |
| BONUS −1 (=10 points) | every 50 ticks |
| pick-up of grain | sets both clocks' dividers to 255 (§12) |

Values at the start of every attempt: `pass_cnt=130, bonus_div=1, time_div=1, lift_div=5, duck_div=1, hen_div=1` **[V, from the model's start state]**.

### 3.4 Acceptance timings (milliseconds) [M]

| Quantity | Value |
|---|---|
| walking speed | 16.3 ms/px right (61 px/s), 16.1 ms/px left |
| climbing speed | 14.8 ms/px (67.5 px/s) |
| TIME −1 | 153 ms (900 units ≈ 138 s); BONUS −1 ≈ 765 ms |
| BONUS→score conversion step (level complete) | 2.17 ms per step (100 steps ≈ 217 ms), continuous buzz, see §12 |
| one hen step (4 px) | ≈ 245 ms per hen (16 px/s); one hen is updated every ≈ 46 ms |
| duck update | ≈ 185 ms |
| lift rise | ≈ 39 ms/px (25.6 px/s) |
| **jump** (take-off → landing, same floor) | **793 ms** (4,762 passes); rise 11 px with steps at 38, 75, 101, 129, 160, 190, 225, 256, 294, 341, 378 ms; apex hang ≈ 74 ms; fall steps at 452, 490, 529, 565, 615, 646, 679, 709, 740, 767, 793 ms |
| horizontal speed in the air | 1 px / 22 ms ⇒ 36 px per jump |
| walk off a ledge to the floor 32 px lower | drift 91 ms + free fall 718 ms ≈ 810 ms |
| free fall of 32 px (ms since fall start → player y) | 54@51, 53@88, 52@128, 51@163, 50@195, 49@230, 48@260, 47@302, 46@328, 45@354, 44@382, 43@408, 42@435, 41@454, 40@475, 39@495, 38@514, 37@529, 36@546, 35@559, 34@574, 33@585, 32@597, 31@622, 30@634, 29@645, 28@659, 27@670, 26@681, 25@696, 24@707, 23@718 |
| level-start pause | ≈ 2 s, silent |

---

## 4. Display

### 4.1 Colour model
Every 8×8 cell has one **ink colour** (set bits of the cell) on a black background; there is no other colour state. Black border/background everywhere.
Palette (index → RGB): 0 `#000000`, 1 `#0000D7`, 2 `#D70000`, 3 `#D700D7`, 4 `#00D700`, 5 `#00D7D7`, 6 `#D7D700`, 7 `#D7D7D7`.
The HUD uses white-on-red cells (ink 7 on paper 2) for its labelled rows (§15).

### 4.2 Tile ids and their colours

| id | meaning | cell colour |
|---|---|---|
| 0 | empty | yellow (6) ink on black – only visible when a sprite/lift bitmap is drawn over it |
| 1 | ladder, left half | 3 (magenta) |
| 2 | ladder, right half | 3 |
| 3 | egg (pick-up) | 7 |
| 4 | grain pile (pick-up) | 3 |
| 5 | platform / floor | 4 (green) |
| 6, 7, 8 | other solid tiles (not used by the maps) | 6, 7, 7 |
| ≥ 9 | decoration (the cage in the top-left corner of every level) — solid | 6 (yellow) |

Solid = id ≥ 5. Standing support = id ≥ 5 or a ladder (1, 2). Pick-ups = 3 and 4.
The tile bitmap set (§5.1) also holds the font and HUD glyphs; ids 0..8 are the map tiles.

### 4.3 Drawing rules [R]
* Sprites are 16×16 px, drawn by **OR-ing** their bitmap onto the cell bitmaps; every cell touched afterwards takes the sprite's colour (player and duck = 6, hens = 5).
  Where a sprite leaves, the covered cells are redrawn from the tile map (tile bitmap + tile colour). So colour clash around sprites is part of the look
  (recolour the whole 8×8 cell under a sprite). A different renderer is acceptable for gameplay.
* **Player / duck** are keyed on the **top edge**: top screen line `191 − y`, left pixel `x` (sub-cell shifted by `x & 7`, so up to 3×3 cells are touched).
* **Hens** are keyed on the **bottom edge**: top screen line `176 − y` (bottom line `191 − y`), left pixel `x & 0xF8` (byte aligned; the 4-px half step is baked into the frame-b sprites), §10.4.
* **Lift bars**: not tile based; a 16×4 px solid bar, bitmap all ones, in the colour of the cells it crosses. **The lift is YELLOW** [V, attribute read from the running original]: empty cells carry attribute `06` (yellow ink on black paper; the ink is invisible until a bitmap is drawn there), so the bar shows as yellow (ZX colour 6), not white.
* Draw order is irrelevant except that sprites overwrite the map; hens sharing a ladder column may wipe pixels of each other when one is redrawn (original behaviour).

### 4.4 Geometry cheat sheet
Floors sit at map rows 0, 4, 8, 12, 16. Floor *k* (k = 0..4): hen `y = 8 + 32k` (8, 40, 72, 104, 136); player `y = hen_y + 15` (feet on the floor line; the player's start `y` is 23).
A standing player shows a 1-px black line between the feet and the platform (sprite rows 0 and 15 are blank for the player frames).

---

## 5. Assets

The art and sound are **data assets that are not reproduced in this document**: the bitmaps, the eight map layouts and the tune notes are the original authors' creative content.
They are loaded at start-up from the PNG files in **`spec-new/assets/`**, whose format is defined here.
Anyone supplying their own art in the same format gets the same game mechanically.

| File | Content | Used by |
|---|---|---|
| `assets/tiles.png` | 184 tiles of 8×8 px | §4.2, §15 |
| `assets/player_sprites.png` | 12 frames of 16×16 px (the player: walk right/left, climbing) | §5.2, §9 |
| `assets/duck_sprites.png` | 4 frames of 16×16 px (the mother duck) | §5.2, §11 |
| `assets/hen_sprites.png` | 9 frames of 16×16 px (frame 0 blank, 1..8 hens) | §5.3, §10.4 |
| `assets/layout_0.txt` … `layout_7.txt` | the eight level layouts (`level mod 8`) as text pictures | §5.4, §17 |
| `assets/death_tune.txt` | the 24 notes of the death tune | §16 |

Hen start positions are the table in §10.1 and the lift columns are in §13 (`LIFT_X = [255, 255, 64, 144, 200, 120, 240, 255]`, 255 = no lift); they are not files.

### 5.1 Format
All PNGs are 8-bit RGB. **Every source pixel is drawn as a 4×4 block** (so the files are comfortably visible in a viewer; read a source pixel from the top-left or centre of its block).
A pixel is *set* when it is not black `(0,0,0)`; the colour of a set pixel is only decoration (ZX normal-intensity palette: tile attribute colour, player yellow, hens cyan) and is not data.
Colours of the original (§4): ladders and grain magenta, floors green, eggs and text white, cage and player yellow, hens cyan.

* **`tiles.png`, `player_sprites.png`, `hen_sprites.png`** are sprite sheets. Every cell is surrounded by a **1 source-pixel dark-grey border** `(48,48,48)`, so the pitch is `size + 2` source pixels and the cell
  content starts at `(col·pitch + 1, row·pitch + 1)`. Cells are numbered row-major from 0 (index = tile id / frame number). Unused trailing cells are plain black.
  * tiles: 8×8 cells, **16 per row** (pitch 10), 12 rows, ids 0..183. The tile bitmap is the cell content, **left pixel = most significant bit**. Where several ids look alike (e.g. 0, 6, 7, 9, 10 are blank), the id is what counts.
  * player sprites: 16×16 cells, **4 per row** (pitch 18), 12 frames (3 rows).
  * duck sprites: 16×16 cells, 4 per row, 4 frames (1 row).
  * hen sprites: 16×16 cells, 9 per row (pitch 18), 9 frames (frame 0 blank, 1..8 as §5.3).
* **`layout_N.txt`** (N = `level mod 8`): exactly **21 lines of 32 characters**, **top line = map row 20, bottom line = map row 0** (the floor), column 0 first; a character per map cell:

  | char | tile id | meaning |
  |---|---|---|
  | `.` | 0 | empty |
  | `[` | 1 | ladder, left half |
  | `]` | 2 | ladder, right half |
  | `o` | 3 | egg |
  | `*` | 4 | grain pile |
  | `=` | 5 | floor / platform |
  | `#` | ≥ 9 | cage decoration; the id comes from the cage table below |

  The cage (map rows 16..20, columns 0..3) is identical in all eight layouts; its cells are `#` in the text and have these ids (`0` = a `.` cell; row 20 is the top line):

  | row \ col | 0 | 1 | 2 | 3 |
  |---|---|---|---|---|
  | 20 | 0 | 168 | 169 | 0 |
  | 19 | 170 | 171 | 172 | 173 |
  | 18 | 174 | 175 | 176 | 177 |
  | 17 | 174 | 175 | 176 | 177 |
  | 16 | 178 | 179 | 180 | 181 |

  Tile ids 6, 7, 8 never occur in layouts.
* **`death_tune.txt`**: one note per line `duration_code pitch` (integers); lines starting with `#` are comments; 24 notes.

```python
LAYOUTS = [read_layout(f"assets/layout_{n}.txt") for n in range(8)]       # level mod 8 → 21 rows × 32 ids (row 0 = bottom, i.e. the LAST text line); copied into the player's map at level start
TILES, PLAYER, HEN = ...                                                  # bitmaps cut from the three sheets (PNG) as above: 8 rows of 8 px / 16 rows of 16 px, top row first
HEN_START = ...                                                           # §10.1: first N entries for levels < 24, all five for levels ≥ 24
LIFT_X = [255, 255, 64, 144, 200, 120, 240, 255]                          # §13
DEATH_TUNE = [(int(a), int(b)) for a, b in notes("assets/death_tune.txt")]   # §16
```
An implementation must refuse to start if one of the files is missing.

Tile ids with a role: ASCII glyphs at their ASCII index (text), digit `d` at `0x9E + d` (HUD digits), life icon `0xB6`, HUD labels `0B 0C 0D` ("SCORE"), `0E 0F 10 11` ("PLAYER"),
`1E 9C 9D` ("LEVEL"), `1B 1C 1D` ("BONUS"), `12 13 14` ("TIME") — any glyphs are acceptable if their role is kept.

### 5.2 Player and duck sprites
**Player** (`player_sprites.png`, 12 frames): 0-3 walk right (4 frames) · 4-7 walk left · 8-11 climbing. Player frames use rows 1..14 only (rows 0 and 15 blank).
Player frame = `base + phase` with `base ∈ {0 right, 4 left, 8 climbing}`, `phase ∈ 0..3`; in the air the frame is `base + 1`.
**Duck** (`duck_sprites.png`, 4 frames, all 16 rows used): 0-1 facing right (wing up/down) · 2-3 facing left. Duck frame = `(0 if px >= dux else 2) + flap`.
Erasing a sprite is not a frame: the covered cells are simply redrawn from the map (§4).

### 5.3 Hen sprites (frames 1..8)
1 walk left (a) · 2 walk right (a) · 3 ladder (a) · 4 ladder (b) · 5 walk left (b, body offset +4 px inside the cell) · 6 walk right (b, +4 px) · 7 eating, facing left · 8 eating, facing right.
Frame 0 draws nothing (= erase). Which frame is shown when: §10.4.

### 5.4 Layout constraints
Each layout: exactly 12 eggs (tile 3); grain piles (tile 4) vary per layout — **10, 7, 10, 6, 13, 10, 4 and 38** for layouts 0-7 (layout 7 is deliberately grain-rich); floors on map rows 0,4,8,12,16
with ladders (tiles 1+2 side by side, 16 px wide) joining them, and the cage decoration (ids ≥ 9) in the top-left corner: 18 cells within map rows 16-20, columns 0-3, identical in all eight layouts. Layouts 2-6 have a lift column free of platforms.

---

## 6. State

```python
class S:
    px = 100; py = 23          # player left edge, top edge (y-up)
    phase = 0; base = 0        # animation phase 0..3; 0 right / 4 left / 8 climbing
    pass_cnt = 130
    air = 0                    # 0 ground, 1 drift (4 ticks after walking off an edge), 2 free vertical movement
    hdir = 0                   # horizontal direction in the air: 0, +1, -1
    vdelay = 0; vcount = 0     # vertical step delay (passes) and countdown
    vdir = 0                   # +1 rising, 0 apex, -1 falling
    onlift = 0
    level = 0                  # 0-based level index; shown 1-based
    eggs_left = 12
    score = [0]*6              # decimal digits, [5] = units
    lives = [5]*players        # per player
    bonus = [d0, 0, 0]         # BONUS/10 as 3 digits; shown with a trailing fixed 0
    time = [d0, 0, 0]          # TIME as 3 digits
    bonus_on = 1
    bonus_div = time_div = duck_div = hen_div = 1; lift_div = 5
    hen_slot = 0               # round-robin 0..4, advances BEFORE use
    hens = [[x, y_bottom, state]]*5      # x == 255 → empty slot
    duck = dict(x=8, y=152, dx=0, dy=0, flap=0)
    lift = dict(x=255, y1=3, y2=67)      # x == 255 → none
    rnd_ptr = 0                # index into the random table (§10.2)
    grain_snd = 0              # pick-up arpeggio countdown
    tenk_digit = 0             # last seen 10 000s digit of the score (extra-life trigger)
```
Per player the program keeps (saved between turns): map (collected eggs stay collected), level, eggs_left, score, lives.

---

## 7. Input

Level-triggered: a function is active while its key is held, sampled when the algorithm below asks for it (`pressed(UP)` etc.). PAUSE/EXIT are edge-triggered and checked once per tick body (§8).
While paused no pass, tick, timer or sound advances and the playfield stays displayed. Touch/gamepad input must feed the same seven functions.

---

## 8. Main loop [V]

```python
def one_pass():
    if (pass_cnt & 0x0F) == 0 and grain_snd != 0:            # arpeggio after a pick-up: every 16 passes
        old = grain_snd; grain_snd -= 1
        if (old & 3) == 0: beep(H=(grain_snd & 0x1F) + 6, L=2)
    if air != 0:
        if pass_cnt == 1: beep(H=(~py) & 255, L=1)            # whistle, once per tick, pitch falls as the player rises
        saved = vcount
        airborne()                                            # §9.4
        if saved == 1:                                        # a vertical step just happened
            for (x, y, s) in hens:                            # stop at the first empty slot
                if x == 255: break
                if hen_hit(x, y): DEATH
    pass_cnt = (pass_cnt - 1) & 255
    if pass_cnt != 0: return
    pass_cnt = 130                                            # ---------- tick body ----------
    # PAUSE toggle / EXIT handled here
    if bonus_on:
        bonus_div -= 1
        if bonus_div == 0: bonus_div = 50; timer_tick(BONUS)
    duck_div -= 1
    if duck_div == 0: duck_div = 12; duck_update()            # §11
    lift_div -= 1
    if lift_div == 0:
        lift_div = 2; lift_update()                           # §13
        if onlift:
            py += 1
            if py >= 0xA4: LEVEL_ENDS('crushed')              # 0xA5 in the original, which rests the rider 1 px higher (§13.2)
    pickup()                                                  # §9.5
    time_div -= 1
    if time_div == 0:
        time_div = 10
        if timer_tick(TIME) == 'expired': LEVEL_ENDS('time up')
        beep(4, 2)
    hen_div -= 1
    if hen_div == 0: hen_div = 2 if level >= 32 else 3; hen_update()    # §10
    if air == 0:
        if pressed(JUMP):
            air = 2; vdelay = 0x8C; vcount = 0; vdir = +1; onlift = 0
            if pressed(RIGHT): hdir = +1; base = 0
            elif pressed(LEFT): hdir = -1; base = 4
            else: hdir = 0                                    # base unchanged
            return                                            # no walking in the take-off tick
        turn()
        if base != 8: walk()
        if not onlift: fall_check()
    climb()                                                   # also while airborne: allows grabbing a ladder in mid-air
```
`DEATH` / `LEVEL_ENDS` leave the loop and return to the level driver (§14). A death can occur inside any draw call (§9.6, §10.3, §11).

---

## 9. The player [V]

### 9.1 Walking (every tick, on ground, `base != 8`)
```python
def walk():
    global px, base, phase
    probe = px - 1 if base == 0 else px
    moved = False
    if pressed(LEFT):
        if px != 1 and tile(probe, py) < 5 and tile(probe, py - 8) < 5:
            px -= 1; base = 4; moved = True
            if (px & 3) == 0: beep(0x28, 5)
    elif pressed(RIGHT):          # evaluated in the same way; LEFT wins when both are held
        if px < 0xEE and tile(probe + 16, py) < 5 and tile(probe + 16, py - 8) < 5:
            px += 1; base = 0; moved = True
            if (px & 3) == 0: beep(0x28, 6)
    else:
        phase = 3; moved = True                               # no key: back to the resting frame (phase 3 + 1 = 0)
    if moved: phase = (phase + 1) & 3                         # DEVIATION [A]: the animation only runs while the player really moves
    draw_player(base + phase)     # includes the duck collision test (§9.6)
    if onlift and not (lift.x - 9 <= px <= lift.x + 9):       # walked off the lift
        air = 1; hdir = +1 if base == 0 else -1; vdelay = 4
```
**Walking animation stops against a barrier [A, deliberate deviation from the original]:** when a pressed direction is blocked (wall, map edge, solid tile) the player does not move and `phase` is **not** advanced — the player stays on the current walking frame (frozen, not animated) and the frame is only redrawn. The original keeps advancing `phase` there, so the legs kept moving on the spot. Releasing the keys still returns to the resting frame, and when the player moves again the animation resumes. Nothing else changes: movement, beeps, ladders, jumps and the airborne frame (`base + 1`) work as before, including the bounce off walls/obstacles while in the air. *Facing* `turn()` runs just before `walk` only when `(py+1) & 7 == 0`:
LEFT sets `base = 4` if tiles at (px−8, py) and (px−8, py−8) are < 5 and tile (px−8, py−16) is in 1..8; RIGHT likewise with column px+16 and `base = 0`. Cosmetic, since `walk` sets `base` when it moves.

### 9.2 Falling off an edge (every tick on ground, not on a lift)
```python
def fall_check():
    if (px & 7) != 0 and tile(px + 8, py - 16) not in (1, 2) and tile(px + 8, py - 16) < 5:
        air = 1; hdir = +1 if base == 0 else -1; vdelay = 4
```
(A player whose x is a multiple of 8 never falls.) `air == 1` is the **drift** phase: for 4 ticks the player keeps moving 1 px/tick horizontally without descending; then
`air = 2, hdir = 0, vdelay = 0xFA, vdir = -1` and it falls straight down.

### 9.3 Ladders (`climb`, only when `px & 7 == 0`)
```python
def climb():
    if px & 7: return
    if tile(px, py + 1) == 1 and pressed(UP):
        base = 8; py += 1
        if (py & 3) == 0: beep(0x1E, 0x14)
        animate()
    elif tile(px, py + 1 - 16 - (8 if ((py + 1) & 7) == 0 else 0)) == 1 and pressed(DOWN):
        base = 8; py -= 1
        if (py & 3) == 0: beep(0x1E, 0x15)
        animate()
def animate():
    phase = (phase + 1) & 3; air = 0; draw_player(base + phase)
```
A ladder can only be used when x is a multiple of 8 and the player is aligned with the left ladder tile; 1 px per tick. While `base == 8` walking is disabled until a landing or step-off
(a landing resets 8 → 0; jumping from a ladder keeps `base`).

### 9.4 Jump / fall engine (every pass while `air != 0`)
```python
def airborne():
    if pass_cnt == 1:                                          # once per tick: horizontal movement
        px += hdir
        if px == 0: hdir = +1
        elif px >= 0xEE: hdir = -1                             # bounce at the walls
        for ya in (lift.y1, lift.y2):
            if lift_board(ya): return                          # §13.2
        if air == 1:                                           # drift
            vdelay -= 1
            if vdelay != 0: land_check(); return
            air = 2; hdir = 0; vdelay = 0xFA; vdir = -1; draw_player(base + phase); return
        vstep(); return
    if air != 1: vstep()

def vstep():
    global vcount, vdelay, vdir, py
    vcount -= 1
    if vcount != 0: return
    if vdir == +1 and py >= 0xA7:   py += 1; vdir = -1; d = 0xFA        # hit the ceiling
    elif vdir == +1:
        d = (vdelay + 10) & 255
        if d == 4: vdir = 0; d = 0                                       # 0xFA+10 wrapped: apex reached
    elif vdir == 0: vdir = -1; d = 0xFA                                  # apex pause elapsed
    else:           d = max(vdelay - 10, 0x28)                           # falling accelerates
    vdelay = vcount = d
    ny = py + vdir
    if ny < 0x10: LEVEL_ENDS('fell out of the world')
    py = ny; land_check()

def land_check():
    draw_player(base + 1)                                                # also the duck collision test
    tx, ty = px + 8, py - 16                                             # tile under the middle of the feet
    if vdir != +1 and hdir != 0:                                         # slide-off nudges
        if hdir == +1:
            if (px & 7) >= 3 and tile(tx + 8, ty) == 5: hdir = -hdir
        else:
            if (px & 7) < 4 and tile(tx - 8, ty) == 5: hdir = -hdir
    t = tile(tx, ty)
    if t == 0: return                                                    # keep falling
    if t != 5:
        if t >= 3: return
        neighbour = tile(tx - 8, ty) if t == 1 else tile(tx + 8, ty)     # ladder: only the platform next to it supports
        if neighbour != 5: return
    if ((py + 1) & 7) != 0: return                                       # land only when the feet are exactly on a floor line
    air = 0
    if base == 8: base = 0
```
Properties: a normal jump rises 11 px, hangs, falls back (§3.4); the player can jump up through platforms (no head collision), cannot steer in mid-air (direction fixed at
take-off, bounces at the walls), and can grab a ladder in mid-air when x is a multiple of 8. Falling steps get faster: delay 250, 240, … down to 40 passes.

### 9.5 Pick-ups (every tick)
```python
def pickup():
    t = tile(px + 8, py - 8); idx = ((py - 8) >> 3) * 32 + ((px + 8) >> 3)
    if t == 3:                        # egg
        map[idx] = 0; add_score(10 * (min(level >> 2, 9) + 1))       # ×10 points: 100 on levels 0-3, 200 on 4-7, … 1000 from 36
        grain_snd = 255; eggs_left -= 1
        if eggs_left == 0: LEVEL_COMPLETE
    elif t == 4:                      # grain
        map[idx] = 0; add_score(5)                                   # 50 points
        bonus_div = time_div = 255; grain_snd = 255
```

### 9.6 Hits against the player
```python
def hen_hit(x, y):                    # x, y of the hen; true = player dies
    a = (x + 5) & 255
    if a < px: return False
    if max(a - 13, 0) >= px: return False
    if y >= py: return False
    return ((y + 28) & 255) >= py
    # i.e. px in (hen.x - 8, hen.x + 5] and py in (hen.y, hen.y + 28]

def duck_hit():                       # run at the end of EVERY player/duck draw
    a = min(px + 8, 255)
    if a < duck.x or max(a - 16, 0) >= duck.x: return
    a = (py + 9) & 255
    if a < duck.y or max(a - 18, 0) >= duck.y: return
    DEATH
```
`hen_hit` is evaluated at every hen erase/draw (each hen move) and during vertical steps of an airborne player. Also ending the attempt: crushed on a lift, falling out of the world (`py < 0x10`), time expiry.

---

## 10. Hens [V]

Five slots `(x, y_bottom, state)`; `x == 255` = empty. States: **1 walk left, 2 walk right, 3 climb down, 4 climb up, 7/8 eating (6 + walk state)**.
Hens exist only on levels with index < 8 or ≥ 16 (levels with 8 ≤ index < 16 have the duck instead of hens and `hen_update` returns immediately).

### 10.1 Start positions `(x, y_bottom, state)` per `level mod 8`
Levels < 24 use only the first *N* entries; levels ≥ 24 use all five. The table below is the hen start table `HEN_START[level % 8]` (entries 1..N followed by N+1..5);
`N = [2, 3, 3, 4, 4, 4, 3, 3][level % 8]`. Unused slots are set to `x = 255`.

| layout | N | entries 1..N | entries N+1..5 (levels ≥ 24) |
|---|---|---|---|
| 0 | 2 | (104,136,2) (72,104,1) | (64,72,1) (152,8,2) (72,40,1) |
| 1 | 3 | (16,8,2) (72,136,2) (224,72,2) | (144,72,1) (168,136,1) |
| 2 | 3 | (16,104,1) (232,32,1) (112,128,2) | (100,80,3) (12,40,4) |
| 3 | 4 | (40,8,1) (216,8,2) (216,136,2) (120,136,2) | (88,8,1) |
| 4 | 4 | (16,40,1) (40,72,2) (40,104,1) (168,72,2) | (224,8,2) |
| 5 | 4 | (24,8,1) (24,104,1) (192,40,2) (232,104,2) | (160,72,2) |
| 6 | 3 | (200,136,1) (12,88,4) (188,64,3) | (40,104,2) (12,24,4) |
| 7 | 3 | (124,112,4) (124,48,4) (160,8,1) | (56,72,1) (192,72,2) |

### 10.2 Random numbers [V]
The only source of randomness is a fixed 256-entry table, advanced once per `hen_update` call (also when the chosen slot is empty; not on levels 8-15, which return first):
`rnd_ptr = (rnd_ptr + 1) & 255`, then `r = RND[rnd_ptr]` (so the first value used after `rnd_ptr = 0` is `RND[1]`). `rnd_ptr` is reset to 0 at the start of each attempt.
Only two bits of each entry are used; the table below holds `r & 3` (as a decimal digit) for index 0..255, 64 per line:

```text
0..63   : 3313333121023003321333332102110010003333333333331210132211200320
64..127 : 0010310011132111131121101310312333333311200010111111210321021210
128..191: 2102002030322013321013220123110012203120332112010021001221003301
192..255: 2100123123330121321331020210012031233212301131213230203020230212
```
* bit 0 clear (`r & 1 == 0`) ⇒ `c = 1`: the hen may now consider a ladder / step off a ladder;
* bit 1 set (`r & 2`) ⇒ "down first / left first", clear ⇒ "up first / right first".
Any generator with independent 50 % bits can replace the table, but then hen paths no longer match the original's, which are fully deterministic.

### 10.3 `hen_update` (one hen per call)
```python
def hen_update():
    global rnd_ptr, hen_slot
    if 8 <= level < 16: return
    rnd_ptr = (rnd_ptr + 1) & 255
    r = RND[rnd_ptr]; c = 1 if (r & 1) == 0 else 0
    hen_slot = (hen_slot + 1) % 5
    h = hens[hen_slot]; x, y, s = h
    if x == 255: return
    draw_hen(x, y, 0)                          # erase; includes hen_hit(x, y) → DEATH
    if s >= 7:                                 # was eating: finish
        h.s = s = s - 6
        old_x = x
        if s != 2: x = (x - 8) & 255; h.x = x
        draw_hen(x, y, 0)                      # (erase again at the shifted x, collision test)
        draw_hen(old_x, y, s); return
    if s < 3:                                  # walking
        if (x & 4) == 0:                       # decision point: every 8 px
            if try_eat(h): return
            probe = x + 8 if s == 2 else x - 8
            turn = probe > 255 or probe < 0
            if not turn:
                t = tile(probe, y - 1)         # what is under the next step
                turn = (t == 0) or (t >= 3 and t != 5)      # keep going only over a ladder (1, 2) or platform (5)
            if turn: h.s = s = 3 - s
        x = x + 4 if s == 2 else x - 4         # 4 px per update
        h.x = x; draw_hen(x, y, s)
        if (x & 4) == 0 or c != 1: return      # ladder entry only on the "half" positions
        order = (check_down, check_up) if (r & 2) else (check_up, check_down)
        for f in order:
            if f(h): return                    # first success wins
        return
    # on a ladder (3 down, 4 up)
    if (y & 4) == 0:                           # every 8 px
        probe = y - 8 if s == 3 else y + 16
        if tile(x, probe) != 1: h.s = s = 7 - s            # end of the ladder → reverse
    y = y + 4 if s == 4 else y - 4
    h.y = y; draw_hen(x, y, s)
    if (y & 4) or c != 1: return
    y8 = y - 8
    if r & 2: order = ((x - 8, 1), (x + 16, 2))             # left first
    else:     order = ((x + 16, 2), (x - 8, 1))             # right first
    for (tx, ns) in order:
        if tile(tx, y8) == 5: h.s = ns; return              # a platform beside the ladder lets the hen step off

def check_down(h): 
    if tile(h.x, h.y - 8) == 1: h.s = 3; return True
    return False
def check_up(h):
    if tile(h.x, h.y + 16) == 1: h.s = 4; return True
    return False

def try_eat(h):                                 # grain beside the hen
    x, y, s = h
    a = x + 8
    if a > 255: return False
    if s != 2:
        a -= 16
        if a < 0: return False
    if tile(a, y) != 4: return False
    map[...] = 0                                # remove the grain (no points for hens)
    h.s = ns = s + 6                            # 7 (left) / 8 (right)
    draw_hen(x if ns == 8 else x - 8, y, ns)
    return True
```
Hens never leave ladders except via these checks. A hen's `x` is a multiple of 4 and alternates between ≡ 0 and ≡ 4 (mod 8); on ladders `x ≡ 4 (mod 8)`.

### 10.4 Drawing a hen [V]
```python
def hen_frame(x, y, s):
    a = 3 if s == 4 else s
    if a < 7:
        if a == 3: a += 1 if (y & 4) else 0       # ladder frame b when bit 2 of y is set → 3 or 4
        else:      a += 4 if (x & 4) else 0       # walking frame b when bit 2 of x is set → 1/5 (left), 2/6 (right)
    return a                                      # 0 = draw nothing; 7/8 as is

left_pixel = x & 0xF8        # NOT x
top_line   = 176 - y         # 16 rows; bottom row on line 191 - y
# OR the frame bitmap onto the background (clip at the right edge when x >= 0xF9)
# colour 5 is written to: 1 cell column if (a < 7 and (x & 4) == 0) else 2 columns, starting at column x >> 3;
#                        2 cell rows if (y & 4) == 0 else 3 rows, the lowest being the row that holds line 191 - y
```
* Facing matches travel: state 1 moves left and uses frames 1/5, state 2 moves right and uses 2/6; turning swaps the pair on the next draw.
* Animation is implicit: every 4-px step flips bit 2 of `x` (or `y` on ladders), so the frames alternate each step (≈ 245 ms). No animation counter.
* Frame-b sprites carry the +4 px offset inside their cell; ladder frames have the body in pixels 4..11 of the cell, so a hen on a ladder (x ≡ 4 mod 8) is centred on the 16-px ladder.
* A hen with `y = 8 + 32k` stands exactly on top of floor row `4k`.

### 10.5 Quirk
A collision detected while a hen is *eating* (`try_eat → draw`) corrupts a hen state byte in the original (stack mis-unwind); undefined. Treat it as a normal fatal hit.

---

## 11. The duck [V]

Active from level index 8 on. On levels < 8 it is parked at (8, 152) in the cage and still drawn (harmless): every update resets it there. At level start `x=8, y=152, dx=dy=0, flap=0` **[A for levels ≥ 8]**.

```python
def duck_update():                               # every 12 ticks
    if duck.x != px: duck.dx = clamp(duck.dx + (1 if duck.x < px else -1), -5, 5)
    if duck.y != py: duck.dy = clamp(duck.dy + (1 if duck.y < py else -1), -5, 5)
    duck_hit()                                   # at the old position
    nx = duck.x + duck.dx
    if nx >= 0xEE or nx < 0:                     # wall: undo the step and head away at full speed
        nx = duck.x; duck.dx = -5 if duck.dx > 0 else 5
    duck.x = nx
    ny = duck.y + duck.dy
    if ny >= 0xA6 or ny < 0x14:
        ny = duck.y; duck.dy = -5 if duck.dy > 0 else 5
    duck.y = ny
    if level < 8: duck.x, duck.y = 8, 152
    duck_hit()                                   # at the new position
    duck.flap ^= 1                               # sprite = (0 if px >= duck.x else 2) + flap
```
(The original computes the rebound position as `x − dx` from the already-advanced value, i.e. the pre-step position, then sets the velocity to ±5 away from the wall; the code above is equivalent.)
The duck homes in with acceleration ±1 per update (≈ 5.4 updates/s), top speed 5 px/update, inside x ∈ [0, 0xED], y ∈ [0x14, 0xA5]. The duck is drawn at `y` = top edge, like the player.

---

## 12. Scoring, bonus, time, lives [V]

```python
def add_score(n):                                # n × 10 points
    for _ in range(n):
        i = 4                                    # tens digit; digit 5 is the always-zero units
        while True:
            score[i] += 1
            if score[i] != 10: break
            score[i] = 0; i -= 1                 # carry
    if score[1] != tenk_digit:                   # the 10 000s digit changed → extra life
        tenk_digit = score[1]; lives[player] += 1    # at most 6 life icons are drawn

def timer_tick(which):                           # which = BONUS or TIME
    d = bonus if which == BONUS else time        # 3 digits, [2] = units
    i = 2
    while True:
        d[i] = (d[i] - 1) & 255
        if d[i] != 255: return 'ok'
        i -= 1
        if i < 0:                                # ran off the end of the counter
            if which == BONUS: bonus_on = 0; return 'ok'
            return 'expired'
        d[i + 1] = 9                             # borrow
```
* Egg: `+100 × (1 + min(level >> 2, 9))`. Grain: +50. Hens give no points.
* BONUS starts at `1000 × min(level + 1, 9)` (digits `[min(level+1, 9), 0, 0]`, displayed with a trailing 0), −10 every 50 ticks while `bonus_on`; once exhausted it stays 0.
* TIME starts at `900 − 100 × min(level >> 4, 5)` (digits), −1 every 10 ticks; at 0 the next tick ends the attempt ("OUT OF TIME !", life lost); each decrement beeps (4, 2).
* **Display when exhausted** [V]: the digits are redrawn only after a decrement that did not run off the end. The underflowing tick leaves the raw bytes `255 9 9` in memory but does not redraw,
  so the screen keeps showing `BONUS 0000` (`TIME 000`). A renderer that draws from the counter must show 000 for an exhausted counter.
* On level completion the remaining BONUS converts: `repeat { timer_tick(BONUS); add_score(1); beep(0x1E, 4) } while bonus_on`. The step that exhausts the counter also scores +10 and beeps
  (BONUS digits 099 ⇒ 99 visible decrements + 1 final = 100 steps = 1000 points).
* **Conversion speed and sound** [V, measured on the original]: the loop has **no delay** — it runs flat out. One step costs ≈ 7.6 kT = **2.17 ms** (BONUS redraw + score add/redraw + the blocking beep of
  4 × (26·30+47) = 3,308 T = 0.95 ms at 4.2 kHz). So 100 steps take ≈ 217 ms and a full 9000-point bonus (900 steps) ≈ 1.95 s, ≈ 350× faster than the 765 ms/decrement of normal play.
  **Sound (exact)**: one step = one burst of **4 square-wave cycles at 4.2 kHz (0.95 ms)**, then **silence for the rest of the step (≈ 1.2 ms)**; the next burst starts 2.17 ms after the previous one started. The ear therefore hears a **low buzz at the burst rate ≈ 460 Hz** (timbre = a 44 % duty pulse train), **not a 4.2 kHz tone**. **Do NOT join the bursts into one continuous 4.2 kHz tone and do not stretch them into a longer note** — that is what sounds "high and weird". Synthesise 4 cycles (3,308 T), then ≈ 4,300 T of silence, per step; no minimum duration, no envelope, no per-step gain ramp.
  **Check your output against these numbers** [V, measured burst by burst on the original]: burst start → burst end = 3,340 T (0.95 ms); burst end → next burst start = **4,300 T (1.23 ms, the whole silence)**; burst start → next burst start = 7,650 T (**2.19 ms**, i.e. the silence is *part of* the period, it is not added after it). The signal is therefore **ON 44 % of the time**; at 44.1 kHz that is ≈ 42 samples of 4.2 kHz square wave, then ≈ 54 samples of silence, repeating. A gap of 2 ms or more between bursts (burst + a full 2.17 ms interval) is wrong and sounds like separate clicks.
  **Sound-to-pause ratio** [V, measured from the speaker-port writes of all 100 steps of a 099 conversion]: every step is exactly **8 speaker edges** (4 cycles, half-cycle ≈ 404-413 T ⇒ ≈ 4.2-4.3 kHz) followed by silence. Sound : silence = **3,310 T : 4,340 T ≈ 0.76 : 1** (≈ **43-44 % sound, 56-57 % silence**; 0.95 ms sound, 1.23 ms silence, 2.19 ms period). With a stricter reading (first edge to last edge = 2,885 T) it is 38 % : 62 %.
  Do not pace the steps with a per-step timer of tens of ms: run them every 2.17 ms (all steps due in a frame at once, display updated per frame) and place each burst at its own step time on that 2.17 ms grid (with the gaps).
* **Grain freezes both clocks**: `bonus_div = time_div = 255`, so the next decrement of each happens after 255 ticks (≈ 3.9 s standing, ≈ 5 s moving), then the normal rhythm resumes. Nothing else
  (eggs, jumping, hens, losing window focus) may stop the clocks; only the pause key.
* Lives: 5 per player; a lost attempt takes one; 0 ⇒ that player is out; the game ends when nobody has lives. 12 eggs per level.

---

## 13. Lifts [V]

Layouts with `level mod 8` in {2, 3, 4, 5, 6} have a lift, at x = **64, 144, 200, 120, 240** respectively (`LIFT_X[level % 8]`; 255 = none). Two yellow (attr 6) 16×4 bars at the same x,
initial `y1 = 3` and `y2 = 67`, rising 1 px every 2 ticks.

```python
def lift_update():                               # every 2 ticks
    if lift.x == 255: return
    for bar in (y1, y2):
        bar += 1
        if bar >= 0xA6: bar = 3                   # wrap to the bottom
```
Bars are drawn as 4 rows of all-ones starting at screen line `191 − y` (rising: the bottom line is `191 − y + 3`).

### 13.1 Boarding (checked once per tick while airborne, for each bar)
```python
def lift_board(ya):
    if lift.x == 255: return False
    if not (lift.x - 9 <= px <= lift.x + 9): return False
    if not (ya + 10 <= py <= ya + 15): return False        # player's top edge, 6 values
    duck_hit()
    onlift = 1; air = 0
    py = ya + 16                                             # rider offset (§13.2)
    beep_random()                                            # short random jingle; any 2-6 note blip is acceptable
    return True
```
While `onlift`: the player moves up with the bar (+1 per lift update), is crushed when `py >= 0xA4`, cannot fall (no `fall_check`), and falls (§9.1) when walking out of the x range.
`onlift` is cleared by jumping (and, in a port, may also be cleared on landing; the original never does).

### 13.2 Rider offset
The original rests the rider at `ya + 17`, which leaves 2 black lines between feet and bar (a floor leaves 1: the blank sprite row). **This specification uses `ya + 16`** (exactly 1 line, like a floor)
and shifts the crush limit from `0xA5` to `0xA4` accordingly. All other lift numbers stay as specified.

---

## 14. Game flow [R]

1. **New game**: every player has 5 lives, level 0, 12 eggs, score 0, a fresh copy of layout 0.
2. **Attempt start** (new level or a player's next turn): draw the HUD and the tile map; set BONUS/TIME (§12); load hens (§10.1), lift (§13), duck (§11); player at **(100, 23)** facing right
   (`base=0, phase=0`, on the ground); reset the state of §3.3 and `rnd_ptr = 0`, `grain_snd = 0`; then a **≈ 2 s silent pause** showing only the HUD and the map — the hens, duck and
   player are **not drawn** until the first pass of the main loop. No music at game start or level start. Multi-player: show "PLAYER n" before each turn.
3. **Play**: §8.
4. **Attempt ends** (hen/duck hit, crushed, fell, time up): the death tune (§16) plays **completely** (≈ 5.3 s) while the frozen playfield stays on screen (no black screen; "OUT OF TIME !"
   is shown over it when time ran out); nothing advances during the tune; then a 1-2 s delay. The map (collected eggs stay collected), level, `eggs_left` and score are saved for the player, `lives -= 1`.
   If lives reach 0 show "GAME OVER PLAYER n" (single player: "GAME OVER") briefly. Next player with lives takes over (round robin); his attempt restarts the current level with BONUS/TIME/hens/lift reset.
5. **Level complete** (`eggs_left == 0`) — sequence [V, timings measured on the original from the end of the BONUS conversion, `A68D`-`A653`]:
   1. convert BONUS (§12);
   2. **≈ 1 s pause** (0 → 0.97 s) with the final playfield — player, hens, duck, eggs, map — still frozen on screen;
   3. **shrinking-box wipe** (`A968`, ≈ 1.1 s, 0.97 → 2.05 s): coloured rectangles shrink towards the centre, and after each of its two passes **the whole bitmap is cleared to 0** (attributes are left alone).
      **Afterwards the screen is empty: no player, no hens, no duck, no map, no HUD** — nothing of the finished level may stay visible;
   4. **"LEVEL nn" message** (`A9CE`, ≈ 1.6 s, 2.05 → 3.69 s) on the *empty* screen: a band of four character rows (rows 11-14, attribute `04` = green ink on black) is painted and the text `level NN`
      (NN = the **new** level, 1-based, continues past 99, shown mod 100) is typed in character by character with short delays, then held for a moment. Nothing else is on the screen;
   5. `level += 1`, load layout `level mod 8` with 12 eggs, go to 2 (the new level's HUD and map are drawn from scratch over the blank screen).
   A port must **not** draw the message over the old playfield; the old screen is cleared first.
   (The message routine `A9CE` is also used for "PLAYER n" and "OUT OF TIME !"; the wipe is not.)
6. **Game end**: all players out, or EXIT pressed at any time → return the scores (§1).

---

## 15. HUD [V] (three top character rows; the playfield is rows 3-23)

Cell values are tile ids (§5.1); digit *d* = tile `0x9E + d`. Colours: labelled cells white on red paper; row 1 yellow; separators black.

* **Row 0**: cols 0-2 `0B 0C 0D` (SCORE, white-on-red); cols 3-4 black; the 6 score digits of player *p* (0-based) at cols `5+7p … 10+7p` (white-on-red): player 1 at 5-10, player 2 at 12-17, …
* **Row 1**: yellow; one life icon (tile `0xB6`) per remaining life of player *p* at cols `5+7p+i`, at most 6.
* **Row 2**, ids by column 0..31:
  `0E 0F 10 11 [P] 00 00 1E 9C 9D 00 [L] [L] 00 00 1B 1C 1D 00 [B] [B] [B] [0] 00 00 12 13 14 00 [T] [T] [T]`
  with colours `white-on-red ×5, black ×2, white-on-red ×6, black ×2, white-on-red ×8, black ×2, white-on-red ×7`; `[P]` = current player 1-4, `[L][L]` = (level + 1) mod 100 as two digits,
  `[B][B][B]` = the three BONUS digits followed by a fixed `0` at col 22, `[T][T][T]` = TIME digits at cols 29-31.

---

## 16. Audio [V]/[M]

**Beeper tone** `beep(H, L)`: `L` cycles of a square wave, one cycle = `26·(H or 256) + 47` WU, i.e. frequency = 3,500,000 / (26·H + 47) Hz. **The game stalls for the duration** (included in §3.1 costs).

| Event | H | L |
|---|---|---|
| walk left step (every 4 px) | 0x28 | 5 |
| walk right step | 0x28 | 6 |
| climb up (every 4 px) | 0x1E | 0x14 |
| climb down | 0x1E | 0x15 |
| airborne, once per tick | 255 − py | 1 |
| pick-up arpeggio (§8): every 16 passes while counting down from 255, sounding when `old & 3 == 0` | (cnt & 0x1F) + 6 | 2 |
| TIME decrement | 4 | 2 |
| bonus conversion step: a 0.95 ms burst every 2.17 ms with silence between (≈ 460 Hz buzz, §12) | 0x1E | 4 |
| lift boarding | short random blip | |

* **Keep the pick-up sound LOW**: each burst is only 2 cycles (0.12 ms at H=6 … 0.58 ms at H=37), a click. The ear hears the train of 64 clicks, one every ≈ 3 ms (≈ 330 Hz buzz) for 0.2–0.4 s with a timbre
  sweep. Synthesise the exact pulse train (no minimum duration, no envelope longer than the beep) and place every burst at its simulated time within the pass sequence
  (time = accumulated WU / 3,500,000), not back-to-back. Walking/climbing/air beeps are real tones (≈ 3.2 kHz walking, 4.2 kHz climbing).
* **Tune player**: notes are `(duration_code, pitch)` pairs; duration = `code × 0.1 s + ≈ 20 ms`, pitch = semitones above middle C (`261.63 Hz · 2^(pitch/12)`); the death tune (24 notes, ≈ 5.3 s)
  is played to the end when an attempt is lost. Its notes are an asset (`assets/death_tune.txt`).
  **Silence between notes [V, measured on the original: speaker-port writes of the whole death tune]**: the notes are *not* joined. Each note sounds for `code × 0.1 s` (≈ 0.199 s for the 24 notes of this tune, all code 2) and is followed by a
  **silence of 20.9 ms (≈ 20.8-20.9 ms for every one of the 23 gaps)**, i.e. a note period of ≈ 219.4 ms; first to last sound ≈ 5.24 s. That 20.9 ms is the "+ ≈ 20 ms" in the duration formula — it is **silence after the note, not part of the tone**.
  An implementation that plays notes back to back (or shortens the silence) makes the repeated pitches (e.g. the four equal notes at the start) blend into one long note. Start each note cleanly (no glide or envelope carried over). No other music exists in the game.

---

## 17. Levels

* Layout index = `level mod 8`; 12 eggs per level.
* Level index 0-7: first N hens per §10.1, no duck (parked). 8-15: **no hens, duck active**. 16-23: N hens + duck. 24-31: five hens + duck. ≥ 32: as 24-31 but the hen update runs every 2 ticks (hens ×1.5 faster).
* Egg value `100 × (1 + min(L >> 2, 9))`; BONUS `1000 × min(L+1, 9)`; TIME `900 − 100 × min(L >> 4, 5)`.

---

## 18. Acceptance tests

1. A 4-px hen step takes ≈ 245 ms average; walking 61 px/s; climbing 67 px/s; jump 793 ms with the §3.4 step times (±5 %).
2. After an egg or grain pick-up TIME and BONUS do not change for ≈ 3.9 s (standing) — and at no other time are they frozen.
3. The BONUS display ends on `0000` and TIME on `000` after exhaustion; conversion of a full-but-one bonus (digits 099) yields 1000 points.
4. Hens and duck are absent during the level-start pause; the death tune plays in full over the frozen screen.
5. Hens: facing equals direction of travel; frames alternate on every step; hens never leave a ladder except by §10.3; with `rnd_ptr` reset to 0 the sequence of hen positions for a fixed input trace is reproducible.
6. Lift rider shows exactly one black line above the bar; the crush limit is `py >= 0xA4`.
7. Jump from a ladder keeps the climbing sprite base; a player at x multiple of 8 never falls off a ledge.

---

## 19. Quirks of the original (keep or fix)
1. Collision during a hen's eating draw corrupts state (§10.5) — treat as a normal death.
2. `onlift` is never cleared except by jumping; after leaving a lift by falling and landing the player keeps re-entering the drift state each tick but lands again as soon as the feet touch a floor. Harmless; clearing `onlift` on landing is allowed.
3. Rider offset 17 vs 16 (§13.2): this spec requires 16.
4. Tile reads above y = 168 hit non-map memory in the original; this spec defines them as empty **[A]**.
5. Walking into a barrier: the original keeps animating the legs on the spot; this spec requires the animation to stop (the frame freezes) while the player does not move (§9.1).
