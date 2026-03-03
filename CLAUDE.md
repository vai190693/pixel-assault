# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the Game

Open `index.html` directly in a browser — no build step, no server required:

```bash
open index.html
```

There are no dependencies, no package manager, and no bundler. The entire game ships as a single self-contained HTML file.

## Git Workflow

Always commit and push after meaningful changes. Use conventional commit prefixes:

- `feat:` — new gameplay feature or screen
- `fix:` — bug fix
- `refactor:` — restructuring without behaviour change
- `chore:` — config or tooling

```bash
git add index.html
git commit -m "feat: describe what changed"
git push
```

Remote: `https://github.com/vai190693/pixel-assault` (branch `main`)

## Architecture

Everything lives inside the `<script>` tag of `index.html`, organised into 11 numbered sections in this fixed order:

| # | Section | What it is |
|---|---|---|
| 1 | **CONSTANTS** | `CANVAS_W/H` (800×600), `DIFFICULTY_CONFIG`, `COLORS` palette |
| 2 | **UTILS** | `clamp()`, `circlesOverlap()` |
| 3 | **INPUT** | `Keys` singleton (held/pressed sets), `Mouse` singleton with canvas-coordinate scaling |
| 4 | **AUDIO** | `Audio` object wrapping Web Audio API — all sounds are synthesised, no files |
| 5 | **PARTICLES** | `ParticleSystem` — pooled square particles, always iterated backwards when removing |
| 6 | **SPRITES** | Stateless `draw*` functions (`drawBackground`, `drawPlayer`, `drawEnemy`, `drawBullet`, `drawCrosshair`) — every one wraps in `ctx.save()`/`ctx.restore()` |
| 7 | **ENTITIES** | `Player`, `Enemy`, `Bullet` classes |
| 8 | **WAVE SYSTEM** | `WaveManager` — tracks spawn scheduling and kill counts |
| 9 | **CAMERA** | `Camera` — screen shake with exponential decay |
| 10 | **GAME** | `Game` class — state machine + `requestAnimationFrame` loop |
| 11 | **BOOT** | `window.onload` — canvas sizing, resize handler, loop kickoff |

### State machine

```
MENU  →[start/Enter]→  PLAYING  →[hp≤0]→  GAME_OVER
  ↑                                              |
  └──────────────[MAIN MENU button]─────────────┘
                 [PLAY AGAIN re-calls _startGame]
```

`Game.state` is one of the three string constants in the `STATE` object. Only `PLAYING` runs `_updatePlaying`; `GAME_OVER` renders the world as a frozen backdrop under an overlay.

### Render pipeline (each frame)

```
rAF → loop(dt) → _updatePlaying(dt)
               → _render()
                    ├─ drawBackground()
                    ├─ camera.apply()        ← shake offset starts
                    │    ├─ drawBullet() ×n
                    │    ├─ particles.render()
                    │    ├─ drawEnemy() ×n
                    │    └─ drawPlayer()
                    ├─ camera.restore()      ← shake offset ends
                    ├─ _renderHUD()          ← always steady (includes crosshair)
                    └─ _renderGameOver()     ← only when state === GAME_OVER
```

`dt` is capped at `0.05s` to prevent entity teleportation on tab restore.

### Input gotchas

- **Mouse coordinates** must go through `Mouse._scale()` (uses `getBoundingClientRect` + canvas logical size ratio) — raw `clientX/Y` are wrong at any window size other than 800×600.
- **Click bleed**: `Mouse.consumeClick()` must be called exactly once per frame in menu/game-over render paths so a start-click doesn't also fire a bullet on frame 1.
- `Keys.wasPressed(k)` is self-clearing — it returns `true` once then removes the key. Never call it twice for the same key in one frame.

### Wave scaling

Each wave: `speed × 1.07^(wave-1)`, `fireRate − 0.05s/wave` (floor 0.5s), `baseEnemies + (wave-1) × 3` total kills required. Heavy enemies (20% spawn chance) have 0.6× speed, 80 HP, larger radius, and are worth 200 pts vs 100 pts.

### Adding a new enemy type

1. Add a flag or type string to `Enemy` constructor and `isHeavy`-style branches in `Enemy.update` and `drawEnemy`.
2. Pass the new spawn probability through `WaveManager._edgePos` / spawn block.
3. Add a score value in `_updatePlaying` bullet collision handler.

### Adding a new sound

Call `Audio._play(oscillatorType, startFreq, endFreq, duration, volume)` from any game event. `endFreq` can be `null` for a flat tone.
