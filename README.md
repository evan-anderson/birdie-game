# Birdie's Watermelon Feast

A 60-second arcade game starring Birdie (a poodle / Irish setter mix) sprinting
through pixel-art Cambridge, UK, catching falling food. Built as a **single
self-contained `index.html`** — no dependencies, no build step, no assets.
Open it in any browser, or serve it and share the URL.

```bash
# run locally
python3 -m http.server 8642
# then open http://localhost:8642
```

## How to play

| Input | Action |
|---|---|
| ← → or A/D | Run left/right |
| Space, W, or ↑ | Jump (clears grounded chocolate) |
| Drag (touch) | Steer Birdie toward your finger |
| JUMP button (touch) | Jump |
| M or 🔊 button | Mute |
| Space/Enter/tap | Start / restart |

### Rules

- **60 seconds** to score as much as possible.
- Food falls from the sky: kibble **+10**, treats **+25**, watermelon **+100**,
  golden watermelon **+250** (spawns twice per game, far away, slow-falling —
  edge arrows point to off-screen melons).
- **Chocolate = instant game over** if touched (toxic to dogs!). It also lands
  and sits on the path for ~2s — jump over it or steer around it.
- Landed food can still be eaten for ~2 seconds before it despawns. If an
  edible despawns uneaten, **your combo resets**.
- **Combo**: every 5 consecutive catches raises the score multiplier
  (×1 → ×5, at combo 5/10/15/20).
- **Zoomies**: eating fills the meter (kibble 6%, treat 12%, melon 30%, golden
  100%). When full: 5 seconds of super speed + a magnet that pulls edibles in.
- **Pigeons** fly across and steal airborne kibble/treats.
- Top-5 local **leaderboard** with arcade-style 3-letter initials
  (persisted in `localStorage`), plus a rank title per run.

---

## Architecture

Everything lives in `index.html`: a `<canvas>`, a mute `<button>`, ~130 lines
of CSS/markup, and one `<script>`. The script is organized top-to-bottom in
sections (search for the `// ----------` banners):

1. **Constants & canvas setup** — `W×H = 480×640` logical pixels, CSS-scaled
   to fit the window (`fit()`), `image-rendering: pixelated` for crisp pixels.
2. **Sprite builder** — `makeSprite(rows, palette, scale)` turns an array of
   strings into an offscreen canvas. Each character indexes a palette color;
   `.` (or any unmapped char) is transparent.
3. **Sprite definitions** — Birdie (4 frames + title variant), food, pigeon
   (2 flap frames), cow.
4. **World painting** — `paintWorld()` draws the whole 1920px background once
   into `bgCanvas`.
5. **Audio** — tiny WebAudio synth. `blip(freq, dur, type, vol, delay, slideTo)`
   plays one oscillator note; the `sfx` object maps named sounds to blip
   combos. No audio files. Created lazily on first user gesture.
6. **Persistence** — leaderboard + high score in `localStorage`.
7. **Game state** — globals + the `dog` object; state machine is a string:
   `title | playing | dying | entername | gameover`.
8. **Input** — keyboard listener + pointer events (mouse and touch unified).
9. **Spawning / eating / update** — all game logic ticks in `update(dt)`.
10. **Render** — `render()` draws world-space (camera-translated) then
    screen-space (HUD, overlays) each frame.
11. **Main loop** — fixed `requestAnimationFrame` loop with `dt` clamped to
    50ms (so a background tab doesn't cause a physics explosion on return).

### Coordinate system & camera

- **World space**: x ∈ [0, `WORLD_W` = 1920], y ∈ [0, 640]. The dog, items,
  pigeons, particles, and popups all live in world coordinates.
- **Camera**: `camX` is the left edge of the visible window, smoothly lerped
  toward `dog.x - W/2 + facing*40` and clamped to `[0, WORLD_W - W]`. On the
  title screen it ping-pongs across the world automatically.
- `render()` wraps world drawing in `ctx.translate(-camX, 0)`; HUD/overlays
  are drawn after `ctx.restore()` in screen space.
- Touch steering converts screen → world: `touchTargetX = pointerX + camX`.

### The world map (painted once in `paintWorld()`)

| x range | Zone | Landmarks |
|---|---|---|
| 0–480 | The Backs | clock tower, King's College Chapel, college wall, willow, lamppost + bicycle, KEEP OFF THE GRASS sign |
| 480–960 | Queens' | red-brick Tudor court, **Mathematical Bridge**, second willow |
| 960–1440 | King's Parade | Senate House, Corpus Clock, market stalls, phone box, post box |
| 1440–1920 | Parker's Piece | open green, cows, football, Reality Checkpoint lamppost, University Library tower |

Vertical bands: sky 0–300 → lawn 300–390 → River Cam 390–450 (ends at
x=1450) → bank 450–472 → cobbled towpath 472–640 (the play area).
`GROUND_Y = 612` is where Birdie's feet sit; items land at y=606.

### Key entities (plain object arrays, no classes)

- `items[]` — `{type, x, y, vy, sway, grounded, groundT}`. Falls, sways
  sinusoidally, lands for 2.2s, then despawns (resetting combo if edible).
- `pigeons[]` — `{x, y, dir, sp, carrying, flapT}`. Fly across the camera
  view, steal airborne kibble/treats on contact, despawn off-camera.
- `particles[]` — `{x, y, vx, vy, grav, t, color}` squares. `grav: 0` makes
  them float (hearts, sparkles, zoomies trail).
- `popups[]` — floating score text.
- `eatFx[]` — a caught item lerping into Birdie's mouth while shrinking.
- `dog` — `{x, vx, facing, animT, eatT, eatColor, hopY, hopV}`. `hopY ≤ 0`
  is height above ground (used by both the happy hop and the jump; gravity
  1150 px/s², jump impulse −480 → ~104px apex). **The collision hitbox
  follows `hopY`**, which is what makes jumping over chocolate work.

### Difficulty & spawning

In `spawnRandomItem()` / the spawn timer, everything scales with
`t = elapsed / 60`:

- spawn interval: `0.9s → ~0.38s`
- fall speed: `140 → 330 px/s` (±15%)
- chocolate probability: `0` for the first 5s (grace), then `6% → 14%`
- fixed shares: melon 10%, treat 24%, rest kibble
- items spawn within ±420px of Birdie (melons ±620px, to tempt you to run)
- golden melons spawn at two scheduled times (`goldTimes`), 500–900px away

### Scoring internals

`eatItem()` is the single entry point for every catch: applies the
multiplier, bumps the combo, fills the zoomies meter, triggers the eat
animation/popup/sounds. `multiplier()` derives ×1–×5 from the combo count.
`finishGame(reason)` routes to `entername` (if top-5) or `gameover`.

---

## Extending the game

**Add a food type** — add a sprite, then one entry in `ITEM_TYPES`
(`pts`, `spr`, `color`, `meter`), then a probability branch in
`spawnRandomItem()`. Everything else (collision, eating, popups, HUD)
picks it up automatically. Sound: add to `sfx` named after the type key.

**Add/change scenery** — edit `paintWorld()`; it's plain canvas rects at
world coordinates, painted once. To widen the world, bump `WORLD_W`
(background canvas, camera clamp, and spawn clamps all key off it).

**New Birdie animation** — copy `DOG_A` (24×16 char grid, palette
`DOG_PAL`), tweak rows, `makeSprite(...)`, and pick the frame in
`drawDog()`. Character key: `r` coat, `d` dark shading/ears, `l` light
curls, `k` eye/nose, `p` tongue, `c` collar.

**Tuning knobs** — `DURATION`, jump impulse (−480 in `jump()`), gravity
(1150), run speed (420 / 660 zoomies), zoomies length (5s), ground linger
(2.2s), pigeon frequency (`pigeonTimer`), rank thresholds (`RANKS`).

**localStorage keys** — `birdie-leaderboard` (JSON top-5),
`birdie-highscore` (legacy single best; kept in sync).

## Dev notes

- `.claude/launch.json` defines the dev server (`python3 -m http.server 8642`).
- The rAF loop pauses when the tab is hidden (browser behavior). For
  headless testing you can call `update(1/60)` and `render()` manually from
  the console — all game globals are accessible.
- To ship: any static host works (GitHub Pages, Netlify, itch.io). It's one
  file with zero network requests.
