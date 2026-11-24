# Sisyphus — project notes for Claude

A 3D browser game: Sisyphus pushes a boulder up an endless mountain. No build step,
no dependencies to install. Open `index.html` directly (`file://` works) or serve the
folder (`python3 -m http.server 8000`). It is deployed with GitHub Pages from `main`:
https://jackbackrack.github.io/sisyphos-video-game/

## Files

- `index.html`: the whole game (Three.js r128, plain JS, one `<script>`).
- `record.html`: in-browser sound recorder. Takes are trimmed, normalized and stored in
  IndexedDB (`sisyphus-voice` / `clips`). The game reads them and they replace the
  built-in sound of the same kind. "Download all" saves one `sisyphus-sounds.zip`.
- `vendor/three.min.js`: Three.js r128. Don't upgrade casually; the code relies on r128 APIs.
- `vendor/rock-texture.js` (`ROCK_JPG`, `ROCK_NRM`), `vendor/voice-sounds.js`
  (`window.VOICE` = {grunt, scream, rumble, wheels, hooves}: arrays of base64 audio),
  `vendor/horses.js` (`HORSE_PNG`). Assets are embedded as base64 JS so the game
  works from `file://`. When an asset in `assets/` changes, regenerate its vendor file.
- `assets/`: source images and the bundled sound files (`assets/voice/`).
- `CREDITS.md`: keep up to date when adding outside assets.

## Controls

W/S or ↑/↓ push or walk, A/D or ←/→ turn (while touching the boulder you rotate around
it), Space jumps (rises again after being crushed), Shift ducks, H toggles the hood,
C cycles rides (none / office chair / tricycle), P or Esc pauses, R restarts.

## How it works (index.html)

- **Units:** sim units → meters with `K = 1/30`, vertical exaggeration `VERT = 2.2`.
  Physics runs in 120 Hz substeps.
- **Terrain:** one infinite mountain. Height = `profileAt(u)` + `ridges()`, with
  `uAt(x,z) = z + waveU(x)` as the "uphill" coordinate. Tiles stream in around the
  player (`tileData`, `corner`, `buildTileView`, `updateTiles`).
  - `hS`: the smooth analytic surface used for gameplay steepness.
  - `hM`: the visible faceted mesh height, used for contact (feet, boulder resting).
- **Trails ("bouldering" routes):** chains of zigzag trails (`chainOf`, `growChain`,
  `ensureTrails`). Many routes start up; most fail (too steep, pinch, or dead end), and
  at least one in every group of 3 chains goes on. Hairpins get decoy spurs and cuts
  that are too steep to push on. `LIMIT = 40°` is the steepest pushable ground.
  Generation must stay **deterministic**: each chain uses its own seeded `mulberry`
  stream, so the result doesn't depend on the order tiles are built.
- **Boulder:** a lumpy, asymmetric icosahedron, low poly with softened edges
  (not flat-faced). `restY()` sets its height so it rests on the mesh and never hovers;
  on a ride it rests on the wheels' contact points instead. Rolling boulders can crush the player.
- **Momentum:** running builds speed (up to `RUN_MAX`), and a running start pushes
  harder. A boulder rolling downhill is more deadly.
- **Rendering:** `rockMaterial()` patches a Three.js material through `onBeforeCompile`:
  triplanar rock texture, bump mapping, ambient occlusion, close-range relief, darkening
  under the boulder, and fog blended to the sky. ACES tone mapping, sRGB output, one sun
  with a large shadow map. Steepness is shown only through natural shading, never colour coding.
- **Character:** modelled on Jules: a simple lumpy mass of curly golden hair, a taupe
  jacket with spiky stylised fur trim, a navy shirt with a soap print, a necklace, ripped
  jeans, black boots with blue laces, a green dinosaur backpack, simple hands and an
  optional hood. `fitToBoulder` keeps the head from going into the boulder.
- **Horses:** 2D cut-out horses gallop in the distance, always moving nose first
  (`placeHorse`, `updateHorses`). Their hoof sound loops, and stops when paused.
- **Audio:** `clips` per kind. Grunt and scream are one-shots (`playClip` avoids
  repeating the last one); rumble, wheels and hooves are loops (`loopSound`, smoothed
  with `loopable()`). Takes from `record.html` override the built-ins.
- **Persistence:** localStorage keys `sisyphus.hood`, `sisyphus.ride`, `sisyphus.bestHeight`.

## Working preferences

- The user tests in Chrome and wants changes visible and natural-looking: organic,
  not plasticky or cartoonish.
- Commit and push only when asked. Commit messages are short, plain sentences.
- Check changes in headless Chrome (`--use-angle=swiftshader`), injecting a script
  that captures errors. Note: IndexedDB doesn't work on `file://` in headless mode, so
  test recorder storage over a local HTTP server.
- Bundling new recordings: the user downloads takes from `record.html`. Convert them
  (ffmpeg: one-shots to m4a; loops to 24 kHz mono wav), put them in `assets/voice/`,
  regenerate `vendor/voice-sounds.js`, and update `CREDITS.md`.
