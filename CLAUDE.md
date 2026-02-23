# Potato Simulator - Claude Code Project Guide

## What This Is
A browser-based combat game called "Potato Simulator" — an HTML file (`potato-simulator.html`) plus a `sprites/` directory of PNG images. A potato fights kitchen utensils across 8 levels. The game was co-designed with a kid who drew all the character art by hand with thick black marker on white paper. His drawings were processed into colored transparent PNGs using the pipeline below.

## Tech Stack
- **HTML + JS + CSS** in a single HTML file, with sprite PNGs loaded from `sprites/` directory
- **HTML5 Canvas** for all rendering (backgrounds, sprites, particles, HUD)
- **Web Audio API** for synthesized retro sound effects
- **No dependencies, no build step** — serve over HTTP (GitHub Pages, `python3 -m http.server`, etc.). Opening via `file://` won't work due to browser CORS restrictions on image loading
- **index.html** is a copy of potato-simulator.html for GitHub Pages — always sync after changes

## Game Structure

### 8 Levels
1. The Sink (forks)
2. The Countertop (forks + knives)
3. The Stovetop (knives + spoons)
4. The Microwave (mixed + Whisk boss)
5. MICROWAVE ON!! (mixed + Cleaver boss)
6. THE MIXER!!! (mixed + Whisk + Mixer boss)
7. THE DEEP FRYER (mixed + Whisk + Fryer Basket boss)
8. THE DOUGH COUNTER (mixed + doughballs + Evil Pizza boss + Rolling Pin boss)

### Key Mechanics
- **Combat**: Directional attack on space bar / mobile tap, 18-frame cooldown
- **Allies**: Food allies recruited from pickups, max 5, they wilt (lose 1 HP every 2 sec), nerfed damage so player must still fight
- **Knockback**: All entities get knocked back on hit, decays at 0.7x/frame
- **Health drops**: 30% chance on enemy kill, heals 25 HP, blinks when expiring
- **Wave spawning**: Enemies spawn gradually (batches of 2-3 every 90 frames), bosses spawn immediately
- **Pizza split**: Evil Pizza splits into 6 pizza slices on death, then reforms at 50% HP for round 2 (won't split again)
- **Sound system**: Web Audio API synth sounds for hit, kill, recruit, heal, level-up
- **Between-level heal**: +50 HP between levels

### Enemy Types & Behaviors
- **Fork**: Basic chase with wobble
- **Knife**: Dash attack (4x speed) when 60-200px away, 120-frame cooldown
- **Spoon**: Circles player at 80px radius, attacks in groups
- **Doughball**: Basic enemy (level 8 only)
- **Bosses** (Whisk, Cleaver, Mixer, Fryer, Pizza, Rolling Pin): Periodic AOE special attack every 200 frames with red warning ring, screen shake

### Important Code Sections (search for these comments)
- `// ============ GAME STATE ============` — levels, enemy/ally definitions, player stats
- `// ============ SPRITE IMAGES ============` — sprite loading from `sprites/` directory
- `// ============ CUSTOM SPRITE DRAWINGS ============` — canvas fallback drawings (used when no sprite image exists)
- `// ============ DRAW ENTITY ============` — main entity rendering (tries sprite image first, falls back to canvas drawing, then emoji)
- `function drawPotato(p)` — player rendering
- `function drawBackground(level)` — all 8 level backgrounds drawn with canvas
- `// ============ SOUND SYSTEM (Web Audio) ============` — Web Audio synth
- `function startLevel()` — level initialization
- `const LEVELS = [` — level definitions (enemies, allies, names)
- `const ENEMY_DEFS = {` — enemy stats (hp, speed, damage, size, score)
- `const ALLY_DEFS = {` — ally stats
- `function createPlayer()` — player stats

### Key Tuning Values
- **Ally cap**: Search `allies.length >= 5` — change 5 to adjust max allies
- **Ally wilt rate**: Search `a._wiltTimer >= 120` — lower = faster wilt; `a.hp -= 1` controls HP per tick
- **Health drop chance**: Search `Math.random() < 0.3` — 0.3 = 30%
- **Between-level heal**: Search `player.hp + 50`
- **Player stats**: In `createPlayer()` — hp: 120, speed: 4, damage: 20, attackRange: 80
- **Wave spawn rate**: Search `waveTimer >= 90` — frames between spawn batches

## Sprite Pipeline

All enemy/player sprites are the kid's actual marker drawings, processed into colored transparent PNGs stored in the `sprites/` directory. Here's how to add a new sprite:

### Requirements
- Python with Pillow and scipy: `pip install Pillow scipy`
- Drawing: thick black marker on white paper, one character per sheet
- Face details (eyes, mouth) need thick strokes to survive the threshold

### Processing Script
```python
from PIL import Image, ImageFilter, ImageDraw
import numpy as np
from scipy import ndimage
import base64

def process_and_colorize(input_path, output_name, fill_color, target_h=200):
    img = Image.open(input_path)
    arr = np.array(img)
    gray = np.mean(arr, axis=2)
    threshold = 120
    mask = gray < threshold

    # Remove noise but keep small details inside the drawing
    labeled, nf = ndimage.label(mask)
    sizes = [(np.sum(labeled == i), i) for i in range(1, nf + 1)]
    sizes.sort(reverse=True)
    main_mask = labeled == sizes[0][1]
    rows = np.any(main_mask, axis=1)
    cols = np.any(main_mask, axis=0)
    rmin, rmax = np.where(rows)[0][[0, -1]]
    cmin, cmax = np.where(cols)[0][[0, -1]]
    margin = 50
    for size, idx in sizes:
        if size < 15:
            mask[labeled == idx] = False
            continue
        blob = labeled == idx
        br = np.where(np.any(blob, axis=1))[0]
        bc = np.where(np.any(blob, axis=0))[0]
        if (br[0] < rmin - margin or br[0] > rmax + margin or
            bc[0] < cmin - margin or bc[0] > cmax + margin):
            if size < 200:
                mask[labeled == idx] = False

    # Crop and resize
    pad = 30
    rmin, rmax = max(0, rmin-pad), min(arr.shape[0], rmax+pad)
    cmin, cmax = max(0, cmin-pad), min(arr.shape[1], cmax+pad)
    mask_c = mask[rmin:rmax, cmin:cmax]
    h, w = mask_c.shape
    result = np.zeros((h, w, 4), dtype=np.uint8)
    result[mask_c] = [25, 25, 25, 255]
    sprite = Image.fromarray(result, 'RGBA')
    sprite = sprite.filter(ImageFilter.SMOOTH)
    new_h = target_h
    new_w = max(30, int(new_h * (w / h)))
    sprite = sprite.resize((new_w, new_h), Image.LANCZOS)
    arr_f = np.array(sprite)
    arr_f[arr_f[:,:,3] < 50] = [0, 0, 0, 0]
    sprite = Image.fromarray(arr_f, 'RGBA')

    # Colorize: flood-fill from edges to find outside, fill inside with color
    arr2 = np.array(sprite)
    alpha = arr2[:,:,3]
    is_transparent = alpha < 50
    is_drawn = alpha >= 50
    labeled2, _ = ndimage.label(is_transparent)
    edge_labels = set()
    edge_labels.update(labeled2[0, :].tolist())
    edge_labels.update(labeled2[-1, :].tolist())
    edge_labels.update(labeled2[:, 0].tolist())
    edge_labels.update(labeled2[:, -1].tolist())
    edge_labels.discard(0)
    outside = np.isin(labeled2, list(edge_labels))
    inside = is_transparent & ~outside
    r, g, b = fill_color
    arr2[inside, 0] = r
    arr2[inside, 1] = g
    arr2[inside, 2] = b
    arr2[inside, 3] = 255
    dr, dg, db = max(0, r//4), max(0, g//4), max(0, b//4)
    arr2[is_drawn, 0] = dr
    arr2[is_drawn, 1] = dg
    arr2[is_drawn, 2] = db
    sprite = Image.fromarray(arr2, 'RGBA')
    sprite.save(f'{output_name}.png')

    # Base64 for embedding
    with open(f'{output_name}.png', 'rb') as f:
        b64 = base64.b64encode(f.read()).decode()
    return b64
```

### To add a new sprite to the game:
1. Process the drawing: `process_and_colorize('photo.jpeg', 'name_sprite', (R, G, B))` — saves PNG to current directory
2. Move the PNG to `sprites/name.png` and add `loadSprite('name', 'sprites/name.png');` in the SPRITE IMAGES section
3. Add enemy def in `ENEMY_DEFS`: `name: { emoji: '🔵', hp: 30, speed: 1.5, damage: 8, size: 30, score: 50, name: 'Name' }`
4. Add to a level in `LEVELS` array: `{ type: 'name', count: 3 }`
5. If it's a boss, add it to the boss filter in `spawnEnemies()`: search for `['whisk','cleaver','mixer','fryer','pizza','rolling_pin']`
6. If it's a boss with special AI, add its type to the boss behavior check: search for the same array in `updateEnemies()`

### Current Sprite Colors
- Potato: (210, 170, 100) — warm tan
- Knife: (190, 200, 210) — steel
- Fork: (200, 210, 220) — silver
- Spoon: (200, 210, 215) — silver
- Whisk: (190, 195, 200) — steel
- Cleaver: (195, 200, 210) — steel blue
- Mixer: (190, 195, 205) — steel
- Fryer Basket: (180, 180, 185) — metallic gray
- Pizza: (230, 180, 70) — golden yellow
- Pizza Slice: (230, 180, 70) — golden yellow
- Rolling Pin: (180, 140, 90) — wood brown
- Dough Ball: (235, 220, 180) — creamy beige

## Art Style Notes
- All sprites are a kid's hand-drawn marker art — preserve this aesthetic
- Canvas fallback drawings use `wobblyLine()` and `wobblyEllipse()` helper functions for a hand-drawn look
- Backgrounds are drawn with canvas — each level has a unique kitchen environment
- The game has a sketchy, imperfect, charming style — don't over-polish it

## Mobile Support
- Touch controls: left/right side of screen to move, tap to attack
- On-screen D-pad and attack button
- Canvas auto-resizes to window
