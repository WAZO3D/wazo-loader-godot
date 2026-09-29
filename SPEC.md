# What a WAZO3D loader does

WAZO3D (https://wazo3d.com) is a web-based 3D modeller. Its models reach other programs as binary glTF
(`.glb`). A **WAZO3D loader** reads them with the platform's own glTF importer and adds one thing:
**a picture it already holds is not read, decoded or uploaded to the GPU again.** A hundred buildings
wearing the same brick cost one brick of video memory.

The first loader, for three.js, is https://github.com/WAZO3D/wazo-loader-three — about fifty lines;
read `src/index.ts` there before starting. This repository is the same thing for another platform.

## What WAZO3D writes

- A `.glb`, glTF 2.0. **Metres, Y up**, the model where it stands (a shared model is one mesh node with
  no transform; a full export may have a node tree — both must read).
- **Every image carries its content name**: the lowercase hex SHA-256 of the image's bytes,
  ```json
  { "bufferView": 7, "mimeType": "image/png", "extras": { "wazo": { "picture": "03c0e31d…7f2061" } } }
  ```
  Two models that wear the same picture carry the same name, whatever else differs.
- Materials: `pbrMetallicRoughness.baseColorTexture` (white `baseColorFactor` under a picture), often a
  `normalTexture` (OpenGL convention) with a `scale`. Samplers are `REPEAT` (10497) for a pattern or
  `CLAMP_TO_EDGE` (33071) for a picture used once.
- **The lean `.glb`**: the same file with every named picture as an address instead of bytes,
  `"uri": "https://wazo3d.com/pictures/<name>"` (in the test files: `"uri": "pictures/<name>.png"`, relative to
  the `.glb`), and no `bufferView`. The site serves `/pictures/<name>` for a year, immutable, with
  `Access-Control-Allow-Origin: *`. A loader that reads lean files downloads each picture once.

## What the loader must do

1. Read any `.glb` through the platform's standard glTF importer, as it would without this loader.
2. For every texture whose image has `extras.wazo.picture`, key it by
   `<picture name>|<wrapS>|<wrapT>` (wrap values from the sampler, 10497 when absent) and **hand out the
   engine texture already made for that key** instead of making a new one — across every model the loader
   reads, not only within one file.
3. For a lean file, fetch a picture's address **only the first time** its name is seen.
4. A glTF without picture names (any other glTF) reads exactly as it would without this loader.
5. Offer: how many distinct pictures are held, and a way to let go of them all (dispose / clear).
6. Optional: check a picture's bytes against its name (SHA-256) before sharing it; a mismatch is not shared.

Keep it small and in the platform's own idiom. No dependency on anything of WAZO3D's.

## The test files (`fixtures/`)

Made by WAZO3D's own export. `expected.json` says what a loader must find:

| file | picture |
|---|---|
| `brick_floor.glb`, `brick_box.glb` | the same brick → **one** texture for both |
| `planks_floor.glb` | planks → a texture of its own |
| `*.lean.glb` | pictures as `pictures/<name>.png` → the brick **fetched once** for the two brick files |
| `plain.glb` | no picture names → reads like any glTF, nothing shared, nothing refused |

The tests must show all four, and run in GitHub Actions where the platform allows it.

## What to deliver

- The loader, in this repository's platform idiom (see the issue).
- Tests against `fixtures/`, and a GitHub Actions workflow running them.
- `README.md`: what it is, how to install and use it, the lean `.glb`, a link to wazo3d.com and to
  wazo-loader-three.
- MIT licence (in the repository already).
