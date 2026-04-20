# Wind-Animated Vegetation Shaders for Neverwinter Nights: Enhanced Edition

Custom GLSL shader overrides that add per-vertex wind animation and foliage-specific lighting to NWN:EE vegetation. Drop-in replacements for the stock `vslit_nm` / `fslit_sm_nm` pipeline — no engine changes, no new uniforms, no baked noise textures. All motion and noise is computed on the GPU.

These are not automatic — you will need to edit your custom content models and assign the appropriate `.mtr` material to each trimesh node. The meshes do not reference the shaders directly; they reference a material, and the material is what binds the custom shader via `customshadervs` / `customshaderfs`. Sample materials are included (`sample_grass.mtr`, `sample_plant.mtr`, `sample_leaf.mtr`) — start from these and swap in your own textures. Grass materials go on grass trimeshes, plant materials on plant trimeshes, leaf materials on leaf trimeshes. Applying the wrong pair (e.g. the leaf material on a grass blade) will produce incorrect motion.

## Features

- **Three tuned vegetation types** — separate shader pairs for mesh-based grass (also supports flat-card grass), 3D-mesh plants (bushes, shrubs, reeds), and tree leaf canopies. Grass is optimized for proper mesh geometry, not quads.
- **Global wind** — sourced from the engine's `areaGlobalWind.xy` uniform, with sensible indoor/outdoor defaults when effectively zero
- **Local wind sources** — array of point sources (`windPointSources*`) driven by physics objects and player footsteps; supports compression, push, and turbulence zones
- **Layered sway motion** — primary sway + secondary harmonic + gusts, with quadratic bend profile along the bbox-local Z axis
- **Mesh-thickness compensation** — plant VS scales down Z-length compensation for volumetric meshes so bushes don't shear like blades of grass
- **Per-vertex leaf flutter** — leaves use `gl_VertexID` pseudo-random flutter + vertical lift/drop instead of the cantilever shear model, so foliage behaves like individual leaves rather than a rigid flag
- **Foliage-specific lighting** — wrapped diffuse, subsurface transmittance, height-based AO, canopy-interior occlusion (leaves), chlorophyll tint, wind-driven color variation, rim lighting
- **Sample materials and noise texture included** — three ready-to-use `.mtr` files (`sample_grass.mtr`, `sample_plant.mtr`, `sample_leaf.mtr`) and the `tex_noise_01.dds` noise texture bound to `texture6` for the wind-noise sampling

## Installation

1. Place all `.shd` files and `tex_noise_01.dds` into your module's override or a hak pak
2. Create a `.mtr` file for each vegetation asset — the fastest path is to copy one of the included samples (`sample_grass.mtr`, `sample_plant.mtr`, `sample_leaf.mtr`), rename it to match your asset (e.g. `mygrass01.mtr`), and swap in your own textures. The `.mtr` is what binds the custom shader and noise texture:

   ```
   customshadervs shd_windgrass_vs
   customshaderfs shd_windgrass_fs

   texture0 your_diffuse
   texture1 your_normal
   texture3 your_roughness
   texture6 tex_noise_01

   parameter float fSwaySpeed    2.5
   parameter float fSwayStrength 1.0
   ```

3. In your `.mdl`, set each vegetation trimesh node's `bitmap` field to the **base name of the `.mtr` file** (e.g. `bitmap mygrass01` to bind `mygrass01.mtr`). The engine resolves the bitmap name to the matching `.mtr`, and the `.mtr` is what references the custom shader. Meshes never name the shader directly.
4. Place the `.mtr` file (and your textures) into the same override / hak pak so the engine can find them alongside the `.mdl`.
5. Tune `fSwayStrength` and `fSwaySpeed` per-asset in the `.mtr`. The included samples show reasonable starting values for each vegetation type.

The engine picks up the overrides automatically — no `.2da` edits or script calls required.

## Files

| File | Pair | Purpose |
|------|------|---------|
| `shd_windgrass_vs.shd` | Grass VS | Mesh grass (and flat-card grass) — sway + local wind response + min-standing-angle clamp |
| `shd_windgrass_fs.shd` | Grass FS | Grass lighting — wrapped diffuse, tip brightness, view-dependent transmittance |
| `shd_windplant_vs.shd` | Plant VS | 3D-mesh plants — sway with mesh-thickness detection for volumetric bushes/shrubs |
| `shd_windplant_fs.shd` | Plant FS | Plant lighting — height AO, translucent backlighting, chlorophyll tint |
| `shd_windleaf_vs.shd` | Leaf VS | Tree canopies — per-vertex flutter + vertical lift/drop, canopy occlusion output |
| `shd_windleaf_fs.shd` | Leaf FS | Leaf lighting — canopy-interior darkening, subsurface glow, leaf variation tinting |
| `sample_grass.mtr` | Sample | Reference material wiring for grass |
| `sample_plant.mtr` | Sample | Reference material wiring for plants |
| `sample_leaf.mtr` | Sample | Reference material wiring for leaves |
| `tex_noise_01.dds` | Texture | Tileable noise texture bound to `texture6` and sampled for per-vertex wind variation |

Each shader file is single, self-contained, and overrides a stock Beamdog shader via the engine's `*Override()` function hooks. No shared `inc_*.shd` files — overrides are deliberately kept as drop-in replacements to stay compatible when Beamdog ships new stock shader versions.

## How It Works

Each VS shader inserts a wind-animation block after the stock vertex position transform and before the stock projection/tangent setup, then hands computed bend values to the FS via named varyings. Noise for per-vertex variation is sampled from `texture6` (the included `tex_noise_01`).

```
Stock vertex position transform (trimesh)
        |
Wind animation block (custom)
        |  - Global sway: primary + secondary + gust
        |  - Local wind sources: compression / push / turbulence zones
        |  - Per-vertex variation sampled from noise texture (texUnit6)
        |  - Combined into finalBendDirection + fCombinedStrength
        |  - Quadratic bend profile along bbox-local Z
        |
Projection + tangent setup (stock)
        |
Fragment Shader (custom)
        |  - Stock lighting base
        |  - Wrapped diffuse, subsurface transmittance
        |  - Height AO (grass/plant) or canopy occlusion (leaf)
        |  - Chlorophyll tint + wind-driven color variation
        V
    Final color
```

Grass and plant share the same bend-as-shear model. The leaf VS replaces it with per-vertex flutter — leaves are not rigid cantilevers and should not move like one.

## VS ↔ FS Varyings

The VS/FS pairs must agree on these varyings:

| Pair | Varyings |
|------|----------|
| Grass | `vGrassHeight`, `vWindIntensity` |
| Plant | `vPlantHeight`, `vWindIntensity` |
| Leaf | `vLeafVariation`, `vWindIntensity`, `vCanopyOcclusion` |

Renaming or adding any of these is a two-file change — update both halves of the pair in the same edit.

## Customization

Per-asset sway tuning is done material-side via three parameters:

| Parameter | Purpose |
|-----------|---------|
| `fSwayStrength` | Base amplitude of the bend. Denormalized in-shader against scale factors (leaves use `/43.0`, grass/plants use `/60.0` — leaves need more amplitude per unit of authored sway) |
| `fSwaySpeed` | Animation speed multiplier on the sway phase |
| `fTilesetMeshOffsetZ` | Runtime Z lift applied inside the VS (grass and plant only — not used by the leaf shader). Workaround for the NWN toolset prop painter, see Known Issues |

All three are exposed as standard `.mtr` `parameter float` entries. The included samples show reasonable starting values for grass, plants, and leaves.

### About the "magic numbers" in the shaders

The non-parameter constants inside the shaders (phase-frequency pairs, zone ratios, Z-compensation factor, gust coefficients, lighting curve coefficients, etc.) look arbitrary but they are not noise. The workflow that produced them was:

1. Many vegetation meshes and materials were hand-tuned against an earlier version of these shaders, with authors picking `fSwayStrength` and `fSwaySpeed` values per asset.
2. Later enhancements (secondary sway, gusts, local wind sources, mesh-thickness compensation, foliage lighting, etc.) were added on top.
3. The in-shader constants were chosen so that the previously authored `fSwayStrength` / `fSwaySpeed` values on those existing meshes still produced the intended motion under the new shader.

So the magic numbers exist to preserve the tuning work already baked into authored content. Changing them without revisiting the authored assets will make previously-tuned vegetation look wrong.

## Wind Source Array

Local wind sources are driven by the engine uniform array — e.g. creatures walking through grass, explosions or spell effects rippling through leaves, creature objects disturbing foliage:

| Uniform | Purpose |
|---------|---------|
| `windPointSourcesCount` | Active source count (capped at `MAX_WINDS`) |
| `windPointSourcesPosition[]` | World-space origin of each source |
| `windPointSourcesRadius[]` | Effective radius |
| `windPointSourcesIntensity[]` | Peak strength |
| `windPointSourcesDuration[]` | Total lifetime |
| `windPointSourcesTimeRemaining[]` | Remaining lifetime (for fade-out) |

Used for physics objects, player footsteps, and any scripted disturbance — vegetation bends away from the source based on distance and time-remaining envelope.

## Requirements

- Neverwinter Nights: Enhanced Edition (GLSL shader support)
- Custom content vegetation meshes wired to custom materials for the relevant shader (grass / plant / leaf)

## Known Issues

- **No shared include file** — shared wind logic is duplicated across the three VS shaders by design. A single `inc_wind.shd` would be DRY-er but would break the drop-in override contract; when Beamdog ships a new stock shader, we re-port our wind block rather than maintain a cross-file dependency.
- **Sway scale factors differ between pairs** — `fSwayStrength` denormalizes with `/43.0` in the leaf VS and `/60.0` in grass/plant VS. Leaves need more amplitude per unit of authored sway, hence the split. Content authors re-using sway values across asset types should keep this in mind.
- **Magic numbers are tied to authored content** — see the Customization section. The in-shader constants were chosen to keep existing hand-tuned meshes looking right after later enhancements were layered in. They look arbitrary in isolation; changing them without a content pass will regress tuned assets.
- **`fTilesetMeshOffsetZ` is a toolset workaround, not a lighting feature** — the NWN toolset's prop/placeable painter treats in-tile grass and plant trimeshes as solid geometry and refuses to paint props through them. To work around this, grass and plant meshes built into tileset tiles are authored *below* their intended render position, so the toolset painter effectively ignores them (the meshes appear transparent/absent at author time). At runtime the VS shader adds `fTilesetMeshOffsetZ` back onto `vLocalPosition.z`, lifting the mesh to its correct visible height. Only set this parameter on tileset-embedded vegetation — placeable vegetation should leave it at `0.0`.

## License

This project is licensed under the [MIT License](LICENSE).
