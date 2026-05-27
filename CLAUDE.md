# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a collection of self-contained browser games — no build tools, no dependencies, no server required. Each game is a single `.html` file that can be opened directly in a browser.

## Running the Games

Open any `.html` file directly in a browser (double-click or `start "" "<file>.html"` on Windows).

## Architecture

All games follow the same single-file pattern:
- `<style>` block — dark retro theme using the shared palette below
- `<canvas id="game">` — everything rendered via Canvas 2D API
- `<script>` block — entire game engine inline, no modules, no imports

### Shared Color Palette
All games reuse this dark retro palette:
```
#0d0d1a  background       #11112a  grid lines
#a8dadc  player/cyan      #e94560  accent/red-pink
#16213e  dark navy        #0f3460  mid navy
#ffe066  yellow/bullets   #55cc77  green
#8855ff  purple           #c97d4e  orange-brown
```

### `shooter.html` — Top-Down Survival Shooter
Structured in this order inside `<script>`:
1. **PALETTE & CONSTANTS** — `P` object (all colors), game tuning values
2. **GLOBALS** — `gameState`, `score`, `currentLevel/Wave`, entity arrays
3. **INPUT HANDLER** — `keys{}` map + `mouse{}` object; `mouse.clicked` is a one-frame flag consumed at end of each loop tick
4. **UTILITIES** — `dist`, `overlap`, `drawText`, `drawButton`, `shuffle`, `randomEdgeSpawn`
5. **PARTICLES** — `Particle` class + `spawnDeathExplosion` / `spawnHitSparks` factories
6. **BULLET** — `Bullet` class; `owner: 'player'|'enemy'`
7. **ENEMIES** — `Enemy` base → `TankEnemy`, `FastEnemy`, `RangedEnemy`; all use circle collision
8. **PLAYER** — `Player` class; `draw()` renders a layered top-down soldier with animated legs, tactical vest, helmet with glowing visor, detailed gun
9. **LEVEL DEFINITIONS** — `LEVELS[]` array; each level has `waves[]`; each wave has `{TANK, FAST, RANGED, interval}`
10. **STATE MANAGEMENT** — `initGame`, `resetFull`, `startWave`, `setState`
11. **UPDATE FUNCTIONS** — `updatePlaying(dt)`, `updateLevelComplete(dt)`
12. **DRAW FUNCTIONS** — `drawBackground`, `drawCrosshair`, `drawHUD`, `drawPlaying`, `drawMenu`, `drawLevelComplete`, `drawGameOver`
13. **GAME LOOP** — `requestAnimationFrame`; `dt` capped at 50ms; state switch dispatches update+draw

**State machine:** `MENU → PLAYING → LEVEL_COMPLETE → PLAYING` (repeat) or `→ GAME_OVER`

**Collision:** circle-circle via `overlap(a, b)`. Pairs: player bullets × enemies, enemy bullets × player, enemies × player (melee, continuous damage × dt).

**Wave flow:** `spawnQueue` (shuffled array of type strings) drains via `spawnTimer`; wave ends when `spawnQueue.length === 0 && enemies.length === 0`.

**Screen shake** and **invincibility frames** on player damage; **hit flash** (white) and **health bar** on enemies.

**Player draw** uses `ctx.save/translate/rotate` so everything is in local rotated space (+X = forward/aim direction, ±Y = sides). All animation values (`stride`, `breathe`, `armSwing`, `recoilOff`) are computed from `walkCycle` and `gameTime`.
