# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository

- **GitHub:** `https://github.com/startyourbiznow2023/retro-browser-games`
- **Branch:** `main`
- **Remote:** `origin`
- **Git identity:** `Fery Yadi <startyourbiznow2023@gmail.com>` (set globally)

After any meaningful change, commit and push:
```
git add <file>
git commit -m "descriptive message"
git push
```

## Running the Games

Open any `.html` file directly in a browser — no server, no build step needed.
```
start "" shooter.html
start "" tictactoe.html
```

## Project Structure

| File | Description |
|------|-------------|
| `shooter.html` | Top-down survival shooter — the main game |
| `tictactoe.html` | Two-player Tic Tac Toe |
| `.gitignore` | Ignores editor artifacts (`.vscode/`, `.idea/`, `*.swp`) |

## Shared Conventions

**Single-file pattern:** every game is one `.html` file — inline `<style>`, a single `<canvas id="game">`, and all logic in one `<script>` tag. No modules, no imports, no external assets.

**Color palette** (all games share this dark retro theme):
```
#0d0d1a  background        #111128  grid lines
#a8dadc  player / cyan     #e94560  accent / red-pink / UI title
#16213e  dark navy         #ffe066  yellow / player bullets
#55cc77  green / ranged    #8855ff  purple / fast enemy
#c97d4e  orange / tank     #ff6b6b  enemy bullets
```

---

## `shooter.html` Architecture

Canvas: 800×600. Everything rendered with Canvas 2D API. All UI (menus, HUD, overlays) drawn on canvas — no DOM elements beyond `<canvas>`.

### Script section order

```
1.  PALETTE & CONSTANTS   — P{} color object, CW/CH, speed/rate/radius tuning values
2.  GLOBALS               — gameState, gameTime, score, level/wave counters, entity arrays
3.  INPUT HANDLER         — keys{} map + mouse{x,y,down,clicked}; mouse.clicked is a
                            one-frame flag, consumed at end of each loop tick
4.  UTILITIES             — dist, overlap, drawText, drawButton, buttonHovered,
                            clamp, shuffle, randomEdgeSpawn
5.  PARTICLES             — Particle class + spawnDeathExplosion / spawnHitSparks
6.  BULLET                — Bullet class; owner: 'player' | 'enemy'; square drawn, circle collision
7.  ENEMIES               — Enemy base → TankEnemy, FastEnemy, RangedEnemy
8.  PLAYER                — Player class with layered animated draw()
9.  LEVEL DEFINITIONS     — LEVELS[] array; each entry has waves[]; each wave: {TANK, FAST, RANGED, interval}
10. STATE MANAGEMENT      — initGame, resetFull, startWave, setState
11. UPDATE FUNCTIONS      — updatePlaying(dt), updateLevelComplete(dt)
12. DRAW FUNCTIONS        — drawBackground, drawCrosshair, drawHUD, drawPlaying,
                            drawMenu, drawLevelComplete, drawGameOver
13. GAME LOOP             — rAF; dt capped at 50ms; switch(gameState) dispatches update+draw
```

### State machine
```
MENU → PLAYING → LEVEL_COMPLETE → PLAYING  (repeats per wave/level)
                               ↘ GAME_OVER  (all 5 levels cleared)
       PLAYING → GAME_OVER                  (player.hp ≤ 0)
       GAME_OVER → MENU
```

### Entity arrays
- `enemies[]`, `playerBullets[]`, `enemyBullets[]`, `particles[]`
- Dead entities are filtered out at the end of each `updatePlaying` tick via `.filter(e => e.alive)` / `.filter(p => !p.dead)`

### Collision
All collision is circle-circle: `overlap(a, b)` → `dist(a,b) < a.radius + b.radius`.

Pairs checked each frame:
1. `playerBullets × enemies` — enemy.takeDamage, spawnHitSparks, bullet dies
2. `enemyBullets × player` — player.takeDamage (skipped if invTimer > 0), spawnHitSparks
3. `enemies × player` (melee) — player.takeDamage(dmg × dt) continuously
4. Soft enemy-enemy push-apart (skipped when enemies.length > 27 for performance)

### Enemy types

| Class | Color | HP | Speed | Behavior |
|-------|-------|----|-------|----------|
| `TankEnemy` | `#c97d4e` orange | 80 | 70 | Direct chase |
| `FastEnemy` | `#8855ff` purple | 20 | 210 | Chase + sinusoidal lateral weave (wavePhase randomised per instance) |
| `RangedEnemy` | `#55cc77` green | 35 | 65 | Maintain ~220px range; fires every ~1.8s into `enemyBullets[]` |

All enemies: `hitFlashTimer` (0.1s white flash on hit), health bar above when damaged, `die()` calls `spawnDeathExplosion` and adds to `score`.

### Wave / level system
- `buildSpawnQueue(cfg)` — Fisher-Yates shuffled array of type strings (`'TANK'`, `'FAST'`, `'RANGED'`)
- `spawnTimer` drains each frame; when it hits 0, `spawnQueue.pop()` creates the next enemy at a random screen edge
- Wave ends when `spawnQueue.length === 0 && enemies.length === 0`
- 5 levels × 2 waves each; difficulty escalates via enemy counts and lower spawn intervals

### Player draw — coordinate system
`draw()` uses `ctx.save → translate(x,y) → rotate(angle)` so **+X = forward (toward mouse), ±Y = sides**.

Layers drawn back-to-front in local rotated space:
1. **Shadow** — flattened ellipse
2. **Boots** — dark soles + uppers; each steps independently via `stride1 = sin(walkCycle)*6`, `stride2 = -stride1`
3. **Pants** — dark navy rects with knee highlights; stride at 55% amplitude
4. **Torso base** — `PLAYER_BODY` cyan rect
5. **Tactical vest** — layered armor plates, chest pockets with stitching, lower strap
6. **Belt** — dark leather + gold buckle
7. **Free arm** — swings opposite the stride via `armSwing = sin(walkCycle + π) * 5`
8. **Gun arm + weapon** — fully detailed: grip (color-coded), magazine with window, slide, ejection port, sight rail, rear/front sights (gold front dot), barrel bore, muzzle cap
9. **Shoulder pads** — rounded circles with rivets
10. **Neck**
11. **Helmet** — dome + crest ridge + brim; dark visor with reflection sheen; cyan glowing eyes that pulse via `sin(gameTime * 3.8)`; ear comms with blinking green antenna
12. **Muzzle flash** — cross/star shape (vertical + horizontal bars + center square + white hot core); triggered by `muzzleFlash.life > 0`

Animation values computed each frame:
- `breathe = sin(gameTime * 2.1) * 0.8` — idle body bob
- `bodyBob = moving ? |sin(walkCycle)| * 1.4 : 0` — step impact bob
- `by = breathe - bodyBob` — combined vertical offset applied to torso and above
- `recoilOff = 5 * (recoilTimer / 0.1)` — gun slides back on fire

### HUD layout
- **Score** — top-left, 6-digit zero-padded, `UI_TITLE` red
- **Level / Wave** — top-center
- **Enemy count** — top-right (enemies remaining + spawn queue)
- **HP bar** — bottom-left, 200px, color shifts green → yellow → red by `hp/maxHp`
- **Wave banner** — fades over 2.5s at wave start
- **Crosshair** — gap-crosshair drawn at mouse position; OS cursor hidden via `cursor: none`

### Screen shake + invincibility
- `shake.timer = 0.18; shake.mag = 5` set on player damage; applied as `ctx.translate(shake.x, shake.y)` before all draws
- `player.invTimer = 0.3` on hit; player flickers (blue shield) while active; damage is skipped during this window
