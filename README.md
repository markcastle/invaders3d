# INVADERS://3D

A modern 3D remake of the 1978 arcade classic, built as a **single self-contained HTML file**. Three.js handles the rendering, the Web Audio API synthesises every sound from scratch, and there is no build step, no bundler, and no assets folder — just `space-invaders-3d.html`.

Open it in a browser and play. Deploy it anywhere that can serve a static file.

---

## How to Play

### The objective

Waves of voxel invaders march across the grid, descending one row every time they hit the edge. Shoot them all before they reach your line. Each wave starts lower, marches faster, and fires more often than the last. There is no final wave — this is an arcade game, and the arcade always wins eventually. The question is your score.

### Controls

| Action | Keyboard | Touch |
| ------ | -------- | ----- |
| Move | `←` / `→` or `A` / `D` | Hold ◀ / ▶ buttons |
| Fire | `Space` | Tap FIRE button |
| Start / Retry | `Enter` (or `Space`) | Tap the screen |
| Pause / Resume | `P` | — |

Touch controls appear automatically on devices that support them.

### Scoring

| Target | Points |
| ------ | ------ |
| 🟢 Octopus (bottom two rows) | 10 |
| 🔵 Crab (middle two rows) | 20 |
| 🟣 Squid (top row) | 30 |
| 🟡 Mothership (occasional flyby) | 50 / 100 / 150 / 300 (random) |

The high score persists for the session and is shown centre-top.

### Things worth knowing

- You can only have **two shots in flight** at once, just like the original. Spray-and-pray doesn't work — make them count.

- The three green **bunkers** are destructible cover. Each block takes two hits (yours *or* theirs) before it's gone, dimming after the first. Bunkers are rebuilt every third wave.

- The swarm **speeds up as it thins out**. The last invader is genuinely frantic — this is the original game's famous difficulty curve, recreated faithfully.

- After losing a ship you respawn at centre with **two seconds of invulnerability** (your ship blinks). Use it to reposition, not to stand in traffic.

- The game ends two ways: lose all three ships, or let the swarm descend to your line. The game over screen tells you which fate you met.

---

## Deploying to Cloudflare Pages (web console, no Wrangler)

The whole game is one file, which makes this about a two-minute job. Two routes, both entirely in the dashboard.

### Option A — Direct upload (fastest)

Good for getting it live right now, or for a quick share link.

1. Rename the file to `index.html` (Pages serves this at the root path automatically). Put it in a folder on its own — e.g. a folder called `invaders` containing just `index.html`.

2. Log in to the [Cloudflare dashboard](https://dash.cloudflare.com) and go to **Workers & Pages**.

3. Click **Create** → choose the **Pages** tab → **Upload assets** (sometimes labelled *Direct Upload*).

4. Give the project a name — this becomes your subdomain, e.g. `invaders-3d` → `invaders-3d.pages.dev`.

5. Drag the **folder** (not the bare file) into the upload area, then click **Deploy site**.

6. That's it. Cloudflare gives you the live URL on the success screen. New versions are deployed the same way — upload again and it creates a new deployment with instant rollback available.

### Option B — Git integration (the proper way)

Better if the game lives in a repo and you want deployments to track commits.

1. Push the file to a GitHub or GitLab repository as `index.html` (root of the repo, or in a subfolder if you prefer).

2. In the Cloudflare dashboard, go to **Workers & Pages** → **Create** → **Pages** tab → **Connect to Git**.

3. Authorise Cloudflare against your Git provider if you haven't before, then select the repository and click **Begin setup**.

4. On the build configuration screen:

   - **Framework preset:** `None`
   - **Build command:** leave **empty** — there is nothing to build
   - **Build output directory:** `/` (or the subfolder containing `index.html`)

5. Click **Save and Deploy**. Every push to the production branch now deploys automatically, and every other branch gets its own preview URL — handy if you fork the game to experiment.

### Notes for either route

- No headers, redirects, or functions are needed. There's nothing server-side at all.

- The only external request the page makes is fetching Three.js from `cdnjs.cloudflare.com` — which, pleasingly, means a Cloudflare Pages deployment never leaves Cloudflare's network.

- Custom domains attach under the project's **Custom domains** tab in the usual way if you want something nicer than `*.pages.dev`.

---

## Tech Deep Dive

This section is for the developer who opens the file and wants to know *why* it's built the way it is, not just what each function does. The code is one `<script>` block, organised top-to-bottom in the order things depend on each other: constants → art → renderer/scene → geometry helpers → game objects → audio → state → input → update loops → main loop.

### Why Three.js and not raw WebGL?

The request was "OpenGL or whatever is better for the job" — and for a browser game that ships as one HTML page, Three.js *is* the better job. Raw WebGL means hand-writing shaders, matrix math, attribute buffers, and a render loop before you draw a single triangle. Three.js gives you the same GPU pipeline with a scene graph, materials, lights, and fog for the cost of one `<script src>` tag from a CDN. Version r128 is used deliberately: it's the last release with a plain global-script (non-module) build on cdnjs, which keeps the single-file, no-build-step constraint honest.

### The big idea: 2D game, 3D presentation

Mechanically, this is still a 2D game. Everything that matters — movement, bullets, collisions — happens on the **XZ plane** (X is left/right, Z is depth, with invaders at negative Z marching toward the player at positive Z). The Y axis is purely cosmetic: hover bobs, explosion arcs, camera height.

This was the single most important design decision, because it means collision detection is cheap axis-aligned overlap testing in two dimensions, exactly as the 1978 original did it, while the *presentation* gets the full 3D treatment — perspective camera, depth fog, parallax, lighting. You get the feel of 3D without the pain of 3D physics.

```text
                 camera (0, 21, 27), looking down-forward
                /
   z = -16   🛸  mothership lane
   z = -13   👾 👾 👾 👾 👾   ← swarm spawns here…
   z = ...   👾 👾 👾 👾 👾      …and steps toward +z
   z =  +9   🟩    🟩    🟩   bunkers (and the "landed" line)
   z = +12.5      🔺          player, clamped to x ∈ [-16, +16]
```

### Rendering the invaders: voxels without the draw-call tax

Each invader species is defined as a **two-frame ASCII bitmap** — literally the original arcade sprites, transcribed as strings:

```js
crab: [[
  '..#.....#..',
  '...#...#...',
  '..#######..',
  ...
```

The naive way to render these in 3D is one small cube mesh per `#`. A crab is ~45 pixels; 50 invaders would mean **over 2,000 meshes and 2,000 draw calls per frame**, which would bring a phone GPU to its knees.

Instead, `voxelGeometry()` bakes each bitmap into a **single merged `BufferGeometry`** at load time. It takes a unit `BoxGeometry` as a template, and for every `#` in the pattern it copies the box's vertex positions (offset to the pixel's location), normals, and indices (offset by the running vertex count) into flat arrays. The result: one invader = one geometry = **one draw call**. The whole swarm is 50 draw calls — trivial.

The classic two-frame "wiggle" animation falls out of this almost for free. Both frames of each species are pre-baked into a `GEO` lookup table at startup, and animating is just a geometry pointer swap:

```js
inv.geometry = GEO[inv.userData.art][G.frame];
```

No morph targets, no skeletons, no per-vertex work at runtime. Swapping a geometry reference costs essentially nothing.

### The swarm: one group, classic step logic

All 50 invaders are children of a single `THREE.Group` (`G.swarm`). Moving the swarm means moving the group — individual invaders never change their local position (except the cosmetic Y bob). An invader's world position is always `swarm.position + invader.position`, which keeps the collision math to one addition.

The march itself is **deliberately not smooth**. It's a discrete step on a timer, because the chunky *step-step-step* is the soul of Space Invaders. Each tick:

1. Measure the world-space extents of the *living* invaders (dead ones don't count — the swarm can travel further as its edges are shot away, just like the original).

2. If the next step would cross a field boundary, reverse direction and step **down** (`+z`) instead.

3. Toggle the animation frame and play the next note of the four-note bass march.

4. If the lowest living invader has crossed `z = 9`, they've landed. Game over.

The famous difficulty curve is two lines of code. The step interval interpolates between the wave's base speed and a frantic 60ms based on the *fraction of invaders still alive*:

```js
const ratio = alive.length / (ROWS * COLS);
const interval = THREE.MathUtils.lerp(0.06, G.stepInterval, ratio);
```

In the original this acceleration was an accident — the hardware simply rendered fewer sprites faster. Here it's recreated on purpose, because the accident *was* the game.

Enemy fire also mirrors the original's rule: only the **bottom-most invader of a column** may shoot. The code buckets living invaders by (rounded) local X, keeps the highest-Z one per bucket, and picks a random shooter from those.

### Collisions: rectangles, not physics

There is no physics engine and no raycasting. Every collision in the game is a 2D axis-aligned box test on the XZ plane:

```js
if (Math.abs(b.position.x - wx) < 1.15 && Math.abs(b.position.z - wz) < 1.0) { ... }
```

The tolerances are tuned slightly generous in the player's favour — arcade games should feel fair-to-good, not pixel-pedantic.

One real-world wrinkle worth knowing about: **tunnelling**. A bullet moving at 38 units/second on a slow frame (the loop clamps `dt` at 50ms) can travel ~1.9 units in a single step — far enough to jump *over* a thin collision window between two frames. The fix is simply making the windows wider than the worst-case step. The bunker check uses a ±1.0 Z window for a block that's visually 0.46 deep, precisely so a laggy phone can't fire through a shield. If you ever crank `BULLET_SPEED`, widen the windows to match.

### Bunkers: destructible cover in 84 cubes

Each bunker is a small ASCII bitmap of individually-tracked cubes. Every block carries `hp: 2` — first hit turns it translucent (a cloned material, so blocks dim independently), second hit hides it. Both your shots and theirs erode them, which creates the classic dilemma: hide behind your shield and you'll eventually shoot a hole in it yourself. Bunkers regenerate every third wave as a small mercy.

### The audio: a synthesiser, not a sample player

There are no audio files. `AudioFX` builds every sound from Web Audio primitives at the moment it's needed:

- **Laser** — a square wave pitch-sweeping 880Hz → 180Hz over 120ms. The sweep is what makes it read as "pew" rather than "beep".

- **Explosions** — a buffer of white noise with a linear decay envelope baked into the samples, pushed through a low-pass filter. The filter cutoff is the character knob: 1400Hz for the crunchy invader pop, 500Hz for the big bassy player death.

- **The march** — the iconic four-note descending bass line (110/98/87/78Hz triangles), advanced one note per swarm step. Because it's tied to the step timer, it accelerates with the swarm automatically — the dread is built in.

- **The mothership** — a 620Hz sine with a second oscillator wired as an **LFO into its frequency** (9Hz, ±120Hz depth), producing the warbling siren. It starts when the saucer spawns and is explicitly stopped when it leaves or dies.

One browser-policy detail that bites everyone the first time: `AudioContext` can't start until a user gesture. `AudioFX.ensure()` lazily creates/resumes the context inside the input handlers, so the first keypress or tap unlocks sound. Before that, every audio call is a silent no-op rather than an error.

### Particles, camera, and the "modern" coat of paint

- **Explosions** are `THREE.Points` clouds — one buffer of positions plus a JS-side array of velocity vectors with gravity applied per frame, faded out via material opacity and removed (and disposed — `Points` geometry/material leak if you don't) after 0.8s. `AdditiveBlending` with `depthWrite: false` is what makes overlapping particles bloom into hot cores instead of z-fighting.

- **The camera** never moves much, and that's intentional. It eases toward `playerX * 0.18` for a subtle parallax that sells the 3D without inducing motion sickness, and a decaying random jitter provides screen shake on player death. A static camera would feel like a render; a chase camera would feel like a different game.

- **Atmosphere** is cheap tricks layered well: scene fog matching the background colour (so the grid floor and starfield dissolve into depth rather than ending), emissive materials standing in for bloom, a `GridHelper` floor for the synthwave horizon, and pure-CSS scanlines and vignette overlaying the canvas for the CRT feel. Total cost: nearly nothing.

### Game state: one machine, one loop

All mutable state lives in a single `G` object with an explicit mode: `title → playing ⇄ paused`, `playing → dying → (playing | gameover)`, `gameover → playing`. The main `requestAnimationFrame` loop dispatches on `G.mode`, so there's exactly one place to look to understand what runs when:

- `dt` is computed per frame and **clamped at 50ms** — without this, returning to a backgrounded tab would deliver a single 30-second timestep and teleport every bullet off the map.

- The `dying` mode exists so that death isn't instant: the explosion plays out, bullets keep flying, and only after the timer does the game either respawn you or roll the game-over screen. Small pause, big feel.

- The title screen runs the swarm at half speed as a background **attract mode**, with two guards: the demo swarm never fires (its bullets would never be updated), and a wrapper around `gameOver` quietly rebuilds the swarm if the demo ever marches itself off the board.

### Tuning it

Everything you'd want to fiddle with is in the constants block at the top of the script: field width, swarm dimensions, speeds, the two-bullet limit. The wave difficulty ramp lives in `buildSwarm()` (start depth, step interval, fire rate per wave). The voxel art is just strings — edit the `ART` bitmaps and the geometry baker does the rest, which makes adding a fourth species about a five-minute job.

### Known trade-offs, honestly stated

- **High score is session-only.** Artifacts and some embedded contexts disallow `localStorage`, so it isn't used. Self-hosted, you could persist it in three lines.

- **The CDN is a dependency.** One external request, to cdnjs. For a fully air-gapped build, inline Three.js into the file (~600KB minified) and it will run from a USB stick.

- **Collision is 2D by design.** If you ever add genuinely 3D mechanics (vertical movement, arcing shots), the plane assumption is the first thing to revisit.

---

*Built with Three.js r128, the Web Audio API, and a healthy respect for Tomohiro Nishikado's original.*
