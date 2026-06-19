# Handoff: Paper Fantasy — an offline pen-and-paper RPG

## Repo / Deployment Status

This folder is now ready to become a standalone static repo.

- `index.html` is the deploy entrypoint for Railway/static hosts and now includes more in-game lore copy.
- `art-evolution.html` is the content-facing old-vs-new art comparison page, including the Claude design -> Codex HTML -> imagegen asset origin trail.
- `art-direction-tests.html` shows the earlier art direction exploration boards.
- `Paper Fantasy.html` is preserved as the original reference prototype.
- `versions/paper-fantasy-reference-2026-06-19.html` is an extra preserved snapshot for rollback/comparison.
- No Node/Express wrapper is required. Railway can deploy static sites from GitHub with zero config when a root `index.html` is present.
- No environment variables are required.

### Railway Deployment

1. Push this folder to a GitHub repo.
2. In Railway, create a new project and choose `Deploy from GitHub repo`.
3. Select the repo. Railway should detect it as a static site from `index.html`.
4. Generate a public domain from the service networking settings.

`.railwayignore` excludes reference material from the deploy upload while keeping it available in git.

### Old Art vs New Art

Open `art-evolution.html` to compare the rough notebook pass against the current manga-ink runtime pass.

- Old rough art lives in `assets/paper-fantasy-*-sketch-rough*.png`.
- Current runtime art lives in `assets/paper-fantasy-*-manga*.png`.
- Older rollback/reference art lives in `assets/archive/`.
- Raw image-generation source files live in `assets/imagegen-sources/`.
- Process/storytelling trail lives in `art-evolution.html`.

### Local Preview

Any static file server works:

```bash
python3 -m http.server 4173
```

Then open `http://localhost:4173/`.

## Overview
**Paper Fantasy** is a single-player, portrait-orientation mobile game for solo travelers killing time offline (planes, trains, no signal). The fiction: it's the sketchbook RPG you secretly play in the margins of your notebook during a high-school lunch break / class. The player explores a hand-drawn dungeon, fights inked monsters in dice-based turn battles, and must periodically **hide the notebook** when the teacher looks over — all rendered to look like graphite, ballpoint, and eraser on lined foolscap paper.

The whole experience is **fully offline** and **auto-saves to `localStorage`** — close the tab mid-run and resume later.

## About the Design Files
The file in this bundle — `Paper Fantasy.html` — is a **design reference + working prototype created in HTML/Canvas**. It is a complete, playable vertical slice that demonstrates the intended look, feel, animation, and game systems. It is **not** intended to be shipped verbatim as your production game.

Your task is to **recreate this design and its systems in the target codebase's environment** using that environment's established patterns:
- If building for **web** (React/Vue/Svelte), keep the `<canvas>` rendering approach (the hand-drawn aesthetic depends on per-frame procedural drawing) but restructure the monolithic IIFE into modules/components and a proper game-state store.
- If building **native** (Swift/Kotlin, Unity, Godot, Flutter `CustomPainter`), port the rendering primitives and game loop to that engine's canvas/draw API.
- If no environment exists yet, a lightweight **TypeScript + Canvas** setup (Vite) or a 2D engine like **Phaser**, **Godot**, or **LÖVE** is the most natural fit — this is a small 2D game, not a DOM/forms app.

The reference is a **single self-contained HTML file** (~900 lines, no build step, no external JS deps; two Google Fonts). Everything is drawn procedurally to a 412×892 logical canvas.

## Fidelity
**High-fidelity.** Final art direction, palette, typography, animation timings, and all game systems are present and tuned. Recreate the visual treatment faithfully — the hand-drawn jitter, the eraser→pencil number animation, and the paper/desk skeuomorphism are the soul of the product. Exact values are documented below and are all readable as constants in the source.

---

## Core Concept & Pillars
1. **It looks hand-drawn, not vectored.** Every line, box, bar, monster, and stat is sketched with deterministic (seeded) jitter so it looks like a human pencil — but does NOT vibrate frame to frame (the jitter is keyed off element position/index, not time).
2. **The Eraser Pipeline.** Numbers (HP/MP/gold/monster-HP) never just swap. When a value changes, an **eraser** rubs the old number out left-to-right (leaving a faint graphite smudge), then a **pencil** rewrites the new value left-to-right. Input is implicitly gated while this plays.
3. **Fog of war.** The dungeon is blank paper; rooms are "drawn in" only as the player's stick-figure steps adjacent to them.
4. **The Teacher Threat.** A real-time meter fills constantly (faster in battle). At 100% the screen flashes "HIDE!" and the player must hit PANIC / swipe down within a countdown or the game ends in **CONFISCATED**.

---

## Technical Architecture (as prototyped)

- **Logical resolution:** `W = 412`, `H = 892` (portrait). Everything is drawn in this coordinate space, then the canvas is scaled with `Math.min(vw/W, vh/H)` to fit the device while preserving aspect ratio. DPR-aware (`canvas.width = W * dpr`, context scaled by `dpr`).
- **Single game loop:** `requestAnimationFrame` → `update(dt)` then `render()`. `dt` is clamped to `0.05s` max.
- **State:** one mutable object `S` holding `scene`, `depth`, player `p`, dungeon `grid`, positions, `battle`, `teacher`, etc. Scenes: `title`, `field`, `battle`, `hide`, `help`, `over`, `win`.
- **Rendering is immediate-mode.** Buttons are NOT DOM — each frame `render()` rebuilds a `buttons[]` array of hit-rects `{id, x, y, w, h}`. Pointer/touch events hit-test against this array (topmost wins). **Reimplement input this way OR replace with real focusable elements for accessibility** (see Accessibility note).
- **No external game libraries.** Audio is synthesized via WebAudio (no audio files).

> ⚠️ Note for a production port: the prototype is a single IIFE with a `window.__pf` debug hook. Break it into: `render/` primitives, `scenes/`, `systems/` (combat, dungeon-gen, teacher, save), and a typed state store.

---

## Screens / Views

### 1. Title (`scene: 'title'`)
- **Purpose:** Entry point. Start new game, continue a save, or read instructions.
- **Layout:** Centered on the paper sheet. Vertically: "PAPER" (graphite) over "FANTASY" (red) wordmark, a sketched dragon, a tagline, a random rumor, then a stack of buttons.
- **Components:**
  - Wordmark "PAPER" — Patrick Hand, 64px, `#3b3a40`, centered.
  - Wordmark "FANTASY" — Patrick Hand, 64px, `#b5331f` (red), centered.
  - Dragon sketch (procedural fallback, see Monsters).
  - Tagline "a dungeon scrawled in the margins" — 20px, `#3b3a40` @ ~55% (graphiteL).
  - Rumor line — Special Elite 13px, italic-feel, 60% graphite.
  - Buttons (sketched rect + label, 240×66, stacked, 80px pitch): `NEW GAME`, `CONTINUE` (only if a save exists), `HOW TO PLAY`.
  - Footer: "✎ plays fully offline · saves itself", 16px graphiteL.
- **Behavior:** `NEW GAME` → `newGame()` → `field`. `CONTINUE` → `loadGame()` (falls back to new). `HOW TO PLAY` → `help` (remembers it came from title).

### 2. Help / How to Play (`scene: 'help'`)
- **Purpose:** Explain controls. Reachable from title and from a `?` button in the field HUD.
- **Layout:** Title "HOW TO PLAY" (38px red) + subtitle, then ~11 rows of `[glyph] [explanation]`, then a back button.
- **Glyphs/lines** cover: movement pad, exit `▼`, chests `▣`, combat `⚔`, the teacher bar `‼`, and autosave `✓`.
- **Behavior:** Back button label is "BACK" if opened from field, "GOT IT — START" if from title. Returns to `_helpFrom` scene.

### 3. Field / Dungeon (`scene: 'field'`) — primary gameplay
- **Purpose:** Explore the dungeon grid, find the exit, trigger encounters and chests.
- **Layout (top → bottom):**
  - **Chalkboard strip** (top, see Teacher HUD) — always visible in field & battle.
  - **Paper sheet** below it (the play surface).
  - Floor title "— floor N —" centered near top of paper.
  - `?` help button top-left of paper (circle + "?", ~34×34 hit area).
  - **Map** — 9×9 grid, cell size `(PAPER.w - 48) / 9 ≈ 40px`, origin at `(PAPER.x+24, PAPER.y+40)`.
  - First-run hint banner (fades out over ~7s of play): "tap ↑↓←→ to walk · reach the ▼ exit" + "watch the TEACHER bar up top!". After it fades, shows the latest `S.msg`.
  - **Stat box** (sketched rect, near bottom): LV, floor, HP bar + value, MP bar + value, gold, potions.
  - **D-pad** — 4 sketched 44×44 squares in a plus arrangement (`↑ ← → ↓`), centered, with "move" label.
  - Rumor line at very bottom (Special Elite 12px, 50% graphite).
- **Map cell rendering:** only cells in `S.explored` are drawn. Each explored floor cell = a hand-drawn rounded-ish rect. Contents: exit `▼` + "exit" label (blue), boss lair = dragon sketch (red), chest `▣`. The player is a **blue stick-figure** (head circle, body, arms, two legs) at the animated position.
- **Movement:** tap a d-pad arrow → if target cell is floor, animate the figure over `0.18s` (linear lerp) to the new cell, leaving a fading graphite "erase puff" at the origin. Walls: small shake + tap sound, no move. On arrival, reveal self + orthogonal neighbors, then resolve the tile (exit/boss/chest/random encounter).

### 4. Battle (`scene: 'battle'`)
- **Purpose:** Turn-based dice combat against one monster.
- **Layout (top → bottom):** chalkboard strip; paper; "!! a wild ink beast appears" (or "!! BOSS") header (red); monster art (centered, ~150px down); monster name; monster HP bar (red); monster HP number (eraser-pipeline animated); a hand-drawn **d6** (shown while player can act / rolling); the battle log line; a compact player stat box (HP + MP bars/values); the **command grid**.
- **Command grid:** 2×2 of sketched buttons (~half-width each, 52px tall, 12/10px gaps):
  - `Attack` — "d6+ATK"
  - `Bolt` — "3MP", blue
  - `Potion` — "x{count}"
  - `Flee` — "~50%" (or "—" disabled vs boss), red
- **Turn flow (state machine on `battle.turn`):** `player` → (Attack/Bolt) sets `rolling` for ~1.05s while the die face cycles → `resolveRoll()` applies damage, screen shake, sound → if monster dead `victoryWait` else `enemyWait` → after a readable delay enemy strikes → back to `player` or `deadWait`. Potion and failed Flee also pass to `enemyWait`. All transitions wait for any active eraser-pipeline animation to finish before proceeding.

### 5. Hide / Cover Sheet (`scene: 'hide'`)
- **Purpose:** The "pretend you're doing homework" screen after PANIC.
- **Layout:** A fake worksheet — "Mathematics — Worksheet 7" heading, red margin line, 6 numbered math problems in faint graphite, footer "…just doing my homework, sir…".
- **Behavior:** Auto-dismisses after `1.6s`, returns to the scene the player was in (`_resume`), and resets the teacher meter/alert.

### 6. Game Over (`scene: 'over'`)
- Two causes: `confiscated` (teacher timer ran out) or `defeated` (HP hit 0 in battle).
- **Confiscated:** "CONFISCATED" (red), "the teacher took your notebook.", plus a rotated red **"SEE ME"** stamp (sketched rect + text, ~-0.18rad).
- **Defeated:** "YOU PASSED OUT", "you fell on floor N.", run summary "Lv X · Yg · Z enemies".
- Dark wash over the paper. `TRY AGAIN` button → title.

### 7. Win (`scene: 'win'`)
- Triggered by defeating the boss (THE FINAL EXAM) or descending past the last floor.
- "★ THE BELL RINGS ★" (red), "you survived the lesson.", run summary, `PLAY AGAIN` → title.

---

## Interactions & Behavior

### Input
- **Touch + mouse:** `pointerdown` hit-tests the `buttons[]` rects; matching `id` routes to `handleBtn(id)`. `Audio2.init()/resume()` is kicked on first interaction (autoplay policy).
- **Swipe-down to hide:** during a teacher alert, a `pointerup` with `dy > 60` and mostly-vertical triggers `panic()`.
- **Keyboard (desktop dev convenience):** arrow keys move; Space/Enter triggers panic during alert.

### Animations & timings (exact)
- **Hand-drawn stroke:** 2 passes (alpha 0.95 then 0.35), segmented by length (~1 seg / 22px), per-segment normal-offset jitter (`amp ≈ 1.1–1.3`). Jitter is `(hash(seed)-0.5)*2` — **seed is derived from element identity, never time**.
- **Move animation:** `0.18s`, linear.
- **Eraser pipeline:** erase phase `0.78s` (eraser sprite sweeps L→R, old digits clipped to the un-erased remainder), then write phase `0.78s` (new digits revealed L→R behind a pencil-tip sprite). Leaves a persistent low-alpha smudge rect behind the number.
- **Die roll:** cycles random faces for ~`1.05s` before resolving.
- **Screen shake:** `shake` set to 4–7 on hits/alerts, decays at `~40/s`, applied as a translated jitter on the whole scene.
- **Toast:** floating red text near bottom, holds for ~2.45s and fades out by ~3.4s.
- **Teacher alert flash:** red wordmark blinks via `sin(time)`.
- **First-run hint:** `S.tut` starts at `7`, decremented by `dt` each field frame; banner alpha = `clamp(tut,0,1)`.

### Audio (synthesized, WebAudio)
A small foley kit, master gain 0.5, mutable via the `♪`/`×` button (top-right):
- `scratch` (pencil) — short bandpass noise burst on writes/moves.
- `erase` — lowpass noise on value changes.
- `tap` — highpass click on button taps / wall bumps.
- `tear` — bandpass noise burst on descend / chest / panic.
- `stinger` — layered saw/square tones on encounter start.
- `hit` — square tone + lowpass noise on damage.
- `levelup` — ascending triangle arpeggio.
- `alarm` — two-tone square on teacher alert.

---

## State Management

### Player (`S.p`)
`hp, maxHp` (start 26), `mp, maxMp` (start 8), `atk` (start 5), `lvl` (1), `kills` (0), `gold` (0), `potions` (2).
**Level-up:** every 3 kills → `lvl++`, `maxHp += 6`, `maxMp += 2`, `atk += 1`, full heal. `levelup` sound.

### Dungeon (`genFloor(depth)`)
- 9×9 grid. Values: `0` wall, `1` floor, `2` stairs/exit, `3` chest, `4` boss lair.
- Generation: random walk of `42 + depth*4` steps from center `(4,4)`. Farthest reachable cell becomes the exit (`2`), or boss lair (`4`) on the final floor (`depth >= MAXD`, `MAXD = 4`). 1–2 chests placed on random remaining floor cells.
- `explored` map seeds with the start cell + neighbors.

### Combat
- **Player attack:** `dmg = d6 + atk`. **Bolt:** costs 3 MP, `dmg = d6 + d6 + atk + 2`.
- **Enemy attack:** `dmg = rand(1..enemy.atk) + floor(depth/2)`.
- **Potion:** +14 HP, consumes one.
- **Flee:** 55% success (disabled vs boss).
- **Victory:** award `enemy.gold`, `kills++`, maybe level-up.

### Teacher Threat (`S.teacher`)
- `_acc` accumulates `dt`; `meter = clamp(_acc * rate, 0, 1)` where rate ≈ `1/34` in field, `1/22` in battle (battle fills faster).
- At `meter >= 1` → `alert = true`, `alertT = 3.6s` countdown, `alarm` sound, board shows a random "problem" (`pickChalk()`).
- If `alertT` reaches 0 with no panic → `over` / `confiscated`.
- `panic()` → `hide` scene for 1.6s, then resume + reset meter/alert.

### Encounters / tiles on arrival
- Exit (`2`) → `descend()`.
- Boss lair (`4`) → `startBattle(boss=true)`.
- Chest (`3`) → 50% gold (`rand(4..10)+depth*3`), 35% +1 potion, 15% +1 ATK.
- Otherwise after step #2, **20% random encounter**.

### Persistence (`localStorage` key `paperfantasy_v2`)
Saves `{v:2, depth, p, grid, hx, hy, explored, steps, scene(field), rumor}`. Saved on: chest, descend, level transitions, every 4th step, panic. `loadGame()` restores into the field scene; battles are not persisted mid-fight (returns to field). **Do not** clear keys you didn't write.

---

## Design Tokens

### Colors (exact)
| Token | Hex / value | Use |
|---|---|---|
| `desk` | `#241f1a` | desk background (bottom of gradient) |
| `deskHi` | `#2f2820` | desk background (top of gradient) |
| `paper` | `#f2ecda` | notebook sheet |
| `paper2` | `#ece4cd` | paper tone blotches |
| `rule` | `rgba(60,96,176,0.30)` | blue horizontal rule lines (26px pitch) |
| `margin` | `rgba(193,57,43,0.55)` | red vertical margin line (at x = paper+30) |
| `graphite` | `#3b3a40` | primary pencil ink |
| `graphiteL` | `rgba(59,58,64,0.55)` | faded pencil / secondary text |
| `smudge` | `rgba(70,68,78,0.16)` | eraser residue |
| `blue` | `#2c3f8c` | ballpoint accents, MP, stick-figure, exit |
| `red` | `#b5331f` | danger, headers, enemy HP, stamps |
| `eraser` | `#e9d4cf` | eraser body |
| `eraserHi` | `#f6e7e3` | eraser highlight |
| `chalk` | `#e9ece4` | chalkboard writing |
| `board` | `#2b3a31` | chalkboard green; frame `#6b5535` |

Threat-meter fill: `>0.7` `#e0563b`, `>0.4` `#e0b13b`, else `#7fae6a`.

### Typography
- **Patrick Hand** (Google Fonts) — the primary "pencil" hand for almost all in-game text. Sizes range 11–64px (wordmarks 64, headers 22–38, body 16–22, small labels 11–16).
- **Caveat** (Google Fonts, 500/700) — chalkboard writing (26px).
- **Special Elite** (Google Fonts) — small "typewriter" meta/footer text (10–13px) and the audio glyph.
- All `pencilText` is drawn with a tiny per-element rotation (`±0.012 rad`) for a hand-placed feel.
- **For offline-first:** self-host/embed these fonts (currently loaded from `fonts.googleapis.com`; cached after first online launch — see Open Items).

### Layout constants
- `PAPER = { x:12, y:80, w:W-24, h:H-92 }`.
- `MAP = { x: PAPER.x+24, y: PAPER.y+40, cell:(PAPER.w-48)/9 }`.
- Chalkboard: `roundRect(8,8,W-16,58,6)`, 5px `#6b5535` border; meter at `(20,50)` size `(W-40)×7`.
- Buttons: d-pad 44×44 (4px gap); battle commands ~half-width × 52px; big menu buttons 240×66.

### Border radius / shadow
- Paper sheet: 3px radius, shadow `rgba(0,0,0,0.45)` blur 18 offsetY 6.
- Chalkboard / cover sheet: 6px radius.
- Most "boxes" are intentionally **not** rounded — they're hand-sketched rects for the sketchbook feel.

---

## Monsters (procedural sketches and raster sprites)
Drawn by `drawMiniMonster(cx,cy,key,seed,col,scale)`. Each `key` is a recipe of sketched circles/lines/dots:
| key | name | tier | hp | atk | gold |
|---|---|---|---|---|---|
| `slime` | Slime Blot | 1 | 8 | 3 | 4 |
| `rat` | Margin Rat | 1 | 6 | 4 | 3 |
| `ghost` | Eraser Wraith | 2 | 12 | 5 | 7 |
| `skull` | Test Skull | 2 | 15 | 6 | 9 |
| `spider` | Ink Spider | 3 | 18 | 7 | 12 |
| `dragon` (boss) | THE FINAL EXAM | 9 | 46 | 9 | 60 |
Stats scale with depth: `hp += depth*2`, `atk += floor(depth*0.6)`, `gold += depth*2`.

## Assets
The original prototype used no image files: all visuals were procedurally drawn to canvas and all audio was synthesized. This repo now includes project-owned imagegen-derived manga-ink raster atlases for the title, monsters, and dungeon props, plus procedural fallbacks:

- `assets/paper-fantasy-title-manga.png` — polished manga-ink title emblem used by the canvas
- `assets/paper-fantasy-monsters-manga-atlas.png` — 3×2 polished manga-ink atlas for `slime`, `rat`, `ghost`, `skull`, `spider`, and `dragon`
- `assets/paper-fantasy-dungeon-manga-atlas.png` — 4×2 polished manga-ink atlas for room tile, chest, exit, hero, die, potion, cover sheet, and pencil sword
- `assets/paper-fantasy-title-sketch-rough.png` — earlier rough raster title emblem, retained as reference material
- `assets/paper-fantasy-monsters-sketch-rough-atlas.png` — earlier rough monster sketch atlas, retained as reference material
- `assets/paper-fantasy-dungeon-sketch-rough-atlas.png` — earlier rough dungeon sketch atlas, retained as reference material
- `assets/paper-fantasy-logo-base.png` — refreshed 512px imagegen-derived manga-ink title emblem/base
- `assets/final-exam-dragon.png` — refreshed 512px imagegen-derived manga-ink dragon cutout
- `assets/imagegen-sources/` — raw built-in `image_gen` PNG outputs used for the active manga pass
- `assets/archive/` — previous runtime PNGs retained for rollback/reference
- `assets/paper-fantasy-logo-base.svg` — editable source for the title emblem
- `assets/final-exam-dragon.svg` — editable source for the dragon

The canvas keeps procedural fallbacks, so the game still runs if an asset fails to load. The only network dependency is the three Google Fonts. Emoji-style glyphs (`▼ ▣ ↑ ↓ ← → ♪ ★ ✎ ✓ ‼`) are rendered as text characters.

---

## Accessibility & Production Notes
- The prototype's canvas-button model has **no semantic accessibility**. For production, either overlay real `<button>`/focusable controls (web) or use the engine's accessibility layer (native). At minimum: larger hit targets are already ≥44px (good); add focus states, screen-reader labels, and a non-time-pressured / colorblind-safe option for the teacher meter.
- **Reduced motion:** offer a toggle that disables screen shake and shortens/【skips the eraser pipeline.
- **`requestAnimationFrame` pauses when the tab/app is backgrounded** — on resume, clamp `dt` (already done) so the teacher meter doesn't jump.
- **Difficulty:** see the design critique — combat is currently simple (d6+ATK). A recommended enhancement is to make long battles spike the teacher meter and add a real combat decision (block/dodge tell, or a push-your-luck reroll). Treat as a product decision, not a bug.

## Files
- `Paper Fantasy.html` — the complete playable prototype (HTML + Canvas + vanilla JS, single file). All constants, the game loop, rendering primitives, asset loading, and systems described above live here. A `window.__pf` object exposes debug hooks (`step`, `btn`, `state`, `fx`) — remove in production.
- `assets/paper-fantasy-title-manga.png` — imagegen-derived polished manga-ink title emblem used at runtime.
- `assets/paper-fantasy-monsters-manga-atlas.png` — imagegen-derived polished manga-ink monster atlas used at runtime.
- `assets/paper-fantasy-dungeon-manga-atlas.png` — imagegen-derived polished manga-ink dungeon/prop atlas used at runtime.
- `assets/paper-fantasy-title-sketch-rough.png` — earlier rough raster title emblem/reference asset.
- `assets/paper-fantasy-monsters-sketch-rough-atlas.png` — earlier rough monster sketch atlas/reference asset.
- `assets/paper-fantasy-dungeon-sketch-rough-atlas.png` — earlier rough dungeon/prop sketch atlas/reference asset.
- `assets/paper-fantasy-logo-base.png` — refreshed 512px manga-ink title logo base/reference asset.
- `assets/final-exam-dragon.png` — refreshed 512px manga-ink boss/title dragon reference asset.
- `assets/imagegen-sources/` — raw built-in `image_gen` PNG outputs used for the current manga pass.
- `assets/archive/` — previous runtime PNGs retained for rollback/reference.
- `assets/paper-fantasy-logo-base.svg` — editable source for the title logo base.
- `assets/final-exam-dragon.svg` — editable source for the boss/title dragon asset.

> Sibling files in the source project (`Pen & Papers.dc.html`, `support.js`) are **earlier exploration / unrelated framework runtime** and are NOT part of this game. Ignore them.
