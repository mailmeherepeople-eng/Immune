# BioCell Chronicles — Handoff Document v2

## What This Is
A single-file HTML5 canvas biology RPG (`index.html`, ~4370 lines). The player is a hematopoietic stem cell that can morph into 8 immune cell types, explore overworld maps (FIELD mode), and fight enemies in a perspective grid battle system (ARENA mode). Mobile-first, landscape-only, tested on iPhone SE (667x375).

## How to Run
- Dev server: `python -m http.server 3000 --directory C:/Users/user/BioGame`
- Open: `http://localhost:3000`
- `requestAnimationFrame` doesn't fire in Claude Preview headless — use a real browser for visual testing.

## File Structure
```
C:\Users\user\BioGame\
├── index.html                    ← THE ENTIRE GAME (~4370 lines)
├── HANDOFF.md                    ← this file
└── assets/
    ├── sprite-manifest.json      ← maps sprite IDs to PNG sheet files
    ├── sprites/
    │   ├── stem_cell.png         ← 768x80, 8 frames in 1 row (96x80 per frame)
    │   └── bacteria.png          ← exists but currently overridden (see DEV flag)
    └── portraits/
        ├── player.png            ← transparent cutout, blue translucent cell girl
        └── marrow_elder.png      ← transparent cutout, old MSC sage with orb
```

---

## What Was Built This Session

### 1. SpriteRenderer System (complete rewrite)
**Location:** ~line 1383, replaces old `drawSpriteAt` + `drawSpriteOnCanvas`

The old system had two separate functions with duplicated logic. Replaced with a unified `SpriteRenderer` object:

```js
SpriteRenderer.draw(ctx, spriteId, x, y, size, frame, dir, anim, tintColor, tintStrength, outlineColor)
SpriteRenderer.drawOnCanvas(canvasEl, spriteId, size, frame, dir, anim, tintColor, tintStrength)
```

**Features:**
- `SPRITE_DEV_OVERRIDE` — top of script, forces all sprites to use one ID during dev
- Color tinting — `source-atop` composite overlays hue on non-transparent pixels only
- Pixel outline — crisp 2px edge via `shadowBlur:2` on offscreen draw
- Single offscreen canvas pool — reused every frame, no GC pressure

**Legacy shims kept** so old `drawSpriteAt`/`drawSpriteOnCanvas` calls still work.

### 2. DEV Sprite Override
**Location:** Line ~821 (top of `<script>`)

```js
const SPRITE_DEV_OVERRIDE = 'stem_cell'; // set null when real sprites are ready
```

When set, ALL sprite draws use this ID regardless of what's requested. This is why bacteria now shows the stem_cell sprite. To restore individual sprites: `const SPRITE_DEV_OVERRIDE = null`.

### 3. Color Tinting Per Character
**MorphData** now has a `tint` field per morph:
- stem_cell: `#a78bfa` (purple)
- neutrophil: `#3b82f6` (blue)
- macrophage: `#7c3aed` (deep purple)
- nk_cell: `#ef4444` (red)
- b_cell: `#06b6d4` (cyan)
- t_cell: `#22c55e` (green)
- dendritic: `#eab308` (yellow)
- eosinophil: `#f97316` (orange)

**EnemyData** now has a `color` field per enemy:
- bacteria/bacteria2: red shades
- virus/virus2: orange shades
- superbug: `#fbbf24` (gold)
- parasite: `#a3e635` (green)
- fungus: `#c084fc` (purple)

### 4. Entity Indicator System
**`drawEntityIndicator(ctx, cx, cy, size, hexColor, pulse)`**

Pulsing glow ellipse drawn UNDER non-player characters (not under the hero). Color = enemy type. Used in:
- FIELD overworld — under encounter tile enemies
- ARENA battle — under enemy sprite

Future mechanic: "disguised enemies" can have mismatched indicator vs sprite color — player has to match the RIGHT cell type to counter them.

### 5. Battle Arena Overhaul
- Grid now fills 92% of canvas width (was capped at `cols × 52px`)
- Non-uniform row heights via quadratic ease — top rows taller on screen so all tiles look square
- `gridWidthTop = gridWidthBottom * 0.80` (subtle taper)
- Bottom edge anchored at 90% of canvas height
- Sprite sizes: player = 68px, enemy = 68px (unified, was mismatched)
- Sprite vertical anchor changed from `y - size + 4` to `y - size * 0.72` so characters sit ON tiles

### 6. Field Sprite Size Equalization
All field sprites now `ts * 1.25`:
- Player: was `ts * 1.6` (hero was too big)
- NPC: was `ts * 1.3`
- Encounter enemy: was `ts * 1.1`

Also removed player `shadowBlur: 6` which was inflating visual footprint vs other characters.

### 7. RPG Dialogue Portrait Fix
- Portraits: `width: 28%` (was 22%), `bottom: 28%` (was 38%)
- Portraits now bleed into text panel like classic JRPGs (Fire Emblem style)
- Text panel padding widened left/right so text doesn't sit under portraits
- `speaking` portrait gets subtle `scale(1.04)` to differentiate speaker

### 8. Test Content Added (REMOVE BEFORE SHIP)
Bone Marrow map now has:
- Bacteria encounter (`1`) at row 5 (3 steps south of spawn)
- Test exit (`E`) at row 7 (5 steps south of spawn)
- `encounters: { '1': 'bacteria' }` marked with `// TEST — remove before ship`

---

## Architecture Overview

### Section 1: Sprite Engine (lines ~820–1510)
- **Pixel helpers**: `px()`, `pxRect()`, `pxCircle()`, `pxOval()`
- **SpriteManifest**: loads PNG sheets from `assets/sprite-manifest.json`. Falls back to procedural SpriteLib.
- **SpriteLib**: procedural pixel art fallback. `_drawHumanoid()` draws 4-directional adventurer sprites.
- **SpriteRenderer**: unified draw system with tinting, outline, offscreen pool (NEW this session)
- **drawEntityIndicator()**: colored glow under non-player chars (NEW this session)
- Legacy shims: `drawSpriteAt()`, `drawSpriteOnCanvas()` still work

### Section 2: Game Data (lines ~1510–1960)
- **MorphData**: 8 cell types, each with `tint` color (NEW), stats, 2 skills, unlock state
- **EnemyData**: 7 enemy types, each with `color` field (NEW), stats, attack patterns
- **UpgradeData**: 12 upgrades
- **MapData**: 4 maps — Bone Marrow, Bloodstream, Lymph Node, Infected Tissue
- **PortraitRegistry / BackgroundRegistry / DialogueData**: RPG dialogue system data

### Section 3: Audio Engine (lines ~1960–2010)
Web Audio API synth. `sfxStep`, `sfxSkill`, `sfxHeal`, `sfxKill`, `sfxMorph`, `sfxLevelUp`, etc.

### Section 4: Game Engine (lines ~2010–4370)
The `Game` object. Key methods:
- `renderOverworld()` — FIELD rendering, 3 passes: floor tiles, walls, then player on top
- `renderBattle()` — ARENA rendering with perspective grid
- `getBattleGrid()` / `getTileCorners()` — perspective math (non-uniform row heights, quadratic ease)
- `showDialogue()` / `renderRpgDialogueLine()` — full-screen RPG dialogue
- `openMorph()` / `openBase()` — overlay panels
- `saveGame()` / `continueGame()` — localStorage

---

## Current Sprite Sizes
| Location | Size |
|---|---|
| Player in FIELD | `tileSize * 1.25` ≈ 65px |
| NPC in FIELD | `tileSize * 1.25` ≈ 65px |
| Encounter enemy in FIELD | `tileSize * 1.25` ≈ 65px |
| Player in ARENA | 68px |
| Enemy in ARENA | 68px |
| Title screen cell | 80px (CSS), DPI-corrected |
| HUD portrait | 28px canvas |
| Morph panel | 48px canvas |

---

## Remaining Issues / TODO

### 🔴 Critical / Must Fix
- `SPRITE_DEV_OVERRIDE` still set — remove or set `null` when individual sprites are ready
- Test bacteria + test exit in Bone Marrow map — remove before ship
- Portrait images for most characters still missing — only `player.png` and `marrow_elder.png` exist
- Most sprite PNGs missing — only `stem_cell.png` and `bacteria.png` exist (bacteria currently overridden)

### 🟡 Gameplay

- **Confetti / screen flash** on correct placement
- **Post-battle dialogue** — no victory/defeat scenes yet
- **Chapter 1 exit narrative** — currently uses generic "CHAPTER 2" message
- **Enemy pre-battle taunt lines** — enemies should say something before arena starts
- **Enemy codex** — biology database fills as you defeat enemies, accessible from base

### 🟠 Story (discussed but not implemented)
The story needs a villain and a soul. Current NPC lines are tutorial tips disguised as dialogue.
- Body needs identity — the player is fighting for *someone*
- Marrow Elder needs a name and a dying arc
- Red Blood Cell should be the comic-relief guide
- Late-game twist: a retrovirus that can corrupt immune memory and speaks to you

### 🟢 Polish / Nice to Have
- Color tinting system is live — can now add `tintStrength` to status effects (poison=green tint, burn=orange tint)
- Sprite outline is live — tweak `outlineColor` per context (white for hero, dark for enemies, gold for boss)
- Arena tile depth edges could use the enemy type color faintly for atmosphere
- Mid-battle morph cooldown UI flicker (rebuildActionButtons every second) — debounce fix pending
- `stem_1cell.png` in sprites folder is a typo/duplicate — delete it

### 🔮 Future Features Discussed
- **Image packs** — sell premium illustration packs for cards/sprites
- **Disguised enemy mechanic** — indicator ring color ≠ sprite appearance, player must identify correctly
- **Memory B-cell system** — tagged enemies stay weaker across saves (immunological memory)
- **Infection spread** — avoid encounter tiles too long and they spread
- **Autoimmune chapter** — friendly fire possible, villain reveals themselves
- **PvP** — Node.js WebSocket server, separate project

---

## How to Add Sprites (when ready)

### Individual sprite PNG
1. Save to `assets/sprites/{morph_id}.png`
2. Add to `sprite-manifest.json`:
```json
"neutrophil": {
  "file": "sprites/neutrophil.png",
  "cols": 8, "rows": 1,
  "artScale": 1.6,
  "anims": { "walk": 0, "idle": 0 }
}
```
3. Set `SPRITE_DEV_OVERRIDE = null` in index.html
4. The tint color is already in MorphData — it applies automatically

### artScale guide
PNG sprites often have whitespace padding. `artScale` zooms the frame so the art fills the same visual space as procedural sprites. Start at `1.5` and tune up.

---

## What to Attach in New Chat
```
@C:\Users\user\BioGame\HANDOFF.md @C:\Users\user\BioGame\index.html — read the handoff, continue building BioCell Chronicles
```
