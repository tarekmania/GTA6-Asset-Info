# 12. Material classes and texture slots

*Part of the [SixNG format specification](README.md). Numbers as of 2026-11-21; confidence: confirmed / inferred / unknown.*

Shader and parameter names are **joaat of the lower-case name** (confirmed):
`standard` = `bcf03fc1`, `diffusetex` = `c8e8c282`, `bumptex` = `1e77bf4c`.
Where the names come from:

- a shader preset list in `common.rpf` names all 344 shaders;
- uncompressed blocks in `shaders.rpf`, before the compressed cache, hold
  NUL-terminated technique, sampler and texture names and 24-byte binding
  records. Variables are `{u32 hash, u32 same hash, 0x30, …}`; texture
  bindings are `{texture hash, parameter hash, 02 01 xx xx, 00 80 00 00}`,
  e.g. (`diffusetexture`, `diffusetex`). The third byte is the parameter's
  slot in the material's texture table (2026-11-23, high; see
  [the shader archive](../structure/06-models-shaders.md#archive-layout-in-detail-2026-11-23)). 88 of 89 sampled drawable
  parameter hashes occur in these records;
- a word search: names not stored as text were found by joining likely
  words and testing joaat. Per-layer sets (`lyr0…`, `lyr1…`) check
  themselves; names that differ only in a final digit do not, because joaat
  states collide before the last character. Single matches of long
  multi-word names are noise and were dropped.

The texture-dictionary key at TXD +0x20 is plain joaat of the texture name
(lower case), as in GTA V.

77 of 229 parameters resolve, about 75 % of uses. 344 shaders over 254 K
drawables (1.83 M shader instances) in `levels.rpf` + `packs.rpf`.

Shader names give the families directly (lit signs and windows, e.g.
`ginos_neon_sign`, use `standard_emissive` in the alpha bucket and
`standard_emissive_layer` as cutout): emissive (`standard_emissive*`,
`signs_emissive`, `*_emissive_layer*`, with an `emissivetex` slot), glass
(`standard_vehicle_glass`, `standard_glass_breakable*`), water
(`standard_alpha_water`, `water_micro_geom`, `standard_alpha_waterfall`),
decals (`standard_decal_hbb`), peds (`ped_standard_skin`) and vehicles
(`standard_vehicle_2lyr`).

## Shader record bytes +0x38..+0x3F

| Byte | Meaning | Confidence |
|---|---|---|
| +0x3C (byte 4) | render-bucket bit mask (below) | confirmed |
| +0x39 (byte 1) | bucket as an index (1 alpha, 2 decal, 3 cutout, 6 water), **+0x80 for instanced / batch-LOD draws** | confirmed |
| +0x3A..+0x3B | u16, maybe the constant-buffer size | unknown |
| +0x3D | always 0xFF | — |

| Bucket bit | Bucket | Instances | Shaders |
|---|---|---|---|
| 0x01 | opaque | 1.09 M | 193 |
| 0x02 | alpha (glass, bottles, VFX, windows) | 32 K | 44 |
| 0x04 | decal | 596 K | 59 |
| 0x08 | cutout (vegetation, cloth, wire) | 87 K | 90 |
| 0x40 | water | 14.6 K | 2 |
| 0x10, 0x20 | one vehicle shader | — | 1 |

**Emissive has no bucket bit**; only the shader family marks it (names
above). A table of known families adds car_paint, window, glass, emissive,
water, terrain_multilayer.

- Unnamed emissive families are recognised by hash, among them `d9ec330c`
  (bulbs) and `faec4fc8` (signs); their colour texture works as the emissive
  texture (inferred from names and usage). The full family list and the
  intensity constant are open (the intensity is probably in the shader
  constant buffers, [§7.6](07-models.md#76-shaders-materials)).
- Vehicle bodies use the car-paint shader `476ac73a` (e.g. `stanierg2`).
  Map wrecks `prop_am_wreck_*` (`levels.rpf`, `.#ft`) use only standard,
  cutout and glass shaders, with their colour in the textures and no runtime
  paint (inferred).

## Texture slots

| Parameter | Name | Holds | Confidence |
|---|---|---|---|
| `c8e8c282` | `diffusetex` | base colour, sRGB | confirmed |
| `1e77bf4c` | `bumptex` | normal, BC1 (B = 255) | confirmed |
| `84c69350`, `58b6e707` | `lyr0normaltex`, `lyr1normaltex` | normal, BC5 | confirmed |
| `27dbd2be`, `7e1f7f44`, `4c759bf1`, `de84c011`, `1a016c7e` | `bumptexture_layer0`–`3`, `tintbumptexture` | terrain layer / detail normals | inferred |
| `c075f9b6` | `lookuptexture` | terrain RGB layer-blend mask | inferred |
| `346394ea`, `7d4f12e2`, `7485dafb` | `lyr0diffusetex`, `lyr1diffusetex`, `dirtdiffusetex` | colour layers of the map blend shaders; `7485dafb` has opacity in alpha. **On the vehicle two-layer shaders `lyr0diffusetex` / `lyr1diffusetex` are linear BC7 data maps, not colour** (tyre tread and sidewall lettering in purple / olive; same mean at mip 0 and small mips), see [§11](11-vehicle-paint.md#vehicle-shaders-2026-11-22) | inferred (map shaders); confirmed not colour (vehicles) |
| `0df47048` | `diffusetex2` | secondary / detail colour, sRGB | inferred |
| `c6f46e79` | `speculartex` | packed: **R** binary mask (metalness?), **G** roughness or gloss, **B** AO / cavity, A unused | inferred |
| `739e3d0d` | `speculartex2` | packed data. On wet-capable surfaces: R region mask, G nearly flat (puddle blobs), B detailed, A AO-like. **Most maps are R = G = 0, B ≈ 0.5**, so B is not roughness: read that way, surfaces turn mirror-like. Treat the channel meaning as unknown | inferred |
| `19df25d7` | ? | vehicle shaders: a **128 × 256 R8G8B8A8 sRGB texture of horizontal colour bands**, shared by all of a car's two-layer materials and the tyre (a later check finds the same texture on every vehicle, first read as a placeholder) ([§11](11-vehicle-paint.md#vehicle-shaders-2026-11-22)); **the shared material palette**: row = a colour record's row ([§11](11-vehicle-paint.md#vehicle-part-colours)) | format and sharing confirmed; role high (2026-11-23) |
| `2c8d7182` | `tintpalettetex` | tint palette **embedded in the model**, on almost every map model: one row of RGBA8 sRGB colours (13 to 2,051, stored linear, padded with `ffbbff`). The texture struct's +0x30 pointer leads to a record whose +8 is the 64-bit store key, so these hashes are not in any `ttpf` store | confirmed |
| `27dbefe1`, `4983edef`, `5bcb1279`, `6df1b6c6`, `d9005af0` | `watersurfacedetailtexture`, `surfacedetailnormal0`–`2`, `surfacedetailiridescence` | water surface detail maps of `water_micro_geom` | names confirmed (joaat) |
| `633aef4d`, `6994ef3f` | `interiordiffusemaskmap` (BC6H), `interiorlayerdiffusemap` | interior maps of `standard_interior` and `standard_2lyr_interior` | names confirmed (joaat) |

**Still unnamed** (2026-11-23): `42110214`, `3ea37b39`, `3022de38`,
`0cd6179f` (four extra per-layer textures of `standard_uber_4lyr_*`, mostly
BC7); `38ebcdb3` (BC1, in the 2-, 3- and 4-layer shaders); `9c362b2c` (an
interior map); `e8f7dc55`, `483cc118`, `2c9a6399`, `e0548ad3`, `d3f807ca`
(`water_micro_geom`); `0cd27763` (in almost every shader, no texture
bound). In the minigame pack all 78 shaders and 65 of 77 texture parameters
are named.

**Placeholder textures** (confirmed). Many building materials point their
packed-data slot (`739e3d0d`) at a 4 × 4 all-zero texture, and some point
the normal slot at a 4 × 4 black one. Read literally, the first gives
AO = 0 (black walls) and the second a broken normal. Ignore textures of
8 px or less in the packed and normal slots. A few materials have only such
placeholders, base colour included (e.g. one shader of
`dn_20_int_hse_14_exterior`); they draw as black boxes and are probably
non-visual proxies (inferred).

## Two-layer multilayer shaders (walls, ground pieces, blend decal)

Shaders `standard_2lyr_pxm` (`b217b263`), `standard_2lyr_pxm_ground`
(`1251cae5`), `standard_2lyr_ground` (`cd8ceb5e`), `standard_2lyr`
(`77d8ab27`) (12 parameters) and `standard_2lyr_pxm_decal_ground`
(`4f0b809b`, terrain blend decal) blend two full texture sets by vertex
colour. Derived from texture sizes and vertex/UV statistics over 4.7 K
materials, not from shader code (as of 2026-11-20).

| Item | 12-parameter family | `4f0b809b` | Confidence |
|---|---|---|---|
| base layer: colour, normal, data, two-channel | p0, p6, p7, p9 | p2, p10, p3, p0 | confirmed (slots of a layer always share an aspect ratio; the layers differ) |
| layer 2: colour, normal, data, two-channel | p4, p10, p5, p2 | p7, p4, p5, p9 | confirmed (same test) |
| overlay (colour + alpha, grime) | p8 | p6 | inferred |
| layer 2 weight | COLOR0.a, the only vertex channel painted over 0..255 | same | inferred |
| layer 2 UV set | TEXCOORD_1 (texel anisotropy matches in 108/117 and 54/72 primitives) | — | inferred |
| base UV set | TEXCOORD_0 (by elimination; TEXCOORD_2 is often constant) | — | inferred, weak |
| COLOR0 r, g, b | r and g unknown ([below](#vertex-colour-red-and-green)); b is the tint palette index ([below](#tint-palettes)) | — | r, g unknown |
| data maps (BC7) | R varies, G about constant per texture, B and A vary | — | unknown meaning |
| two-channel maps (BC1) | R = 0, G and B anti-correlated | — | unknown meaning |

The 17-parameter ground shaders are a different family, four layers by
their names (`standard_uber_4lyr_spx_dm_hbb`, `standard_uber_4lyr_pxm_hbb_fixuv1`,
`standard_uber_4lyr_dpm_dm_hbb`), mapped in the next section.

## Four-layer ground shaders

Shaders `standard_uber_4lyr_dpm_dm_hbb` (`61e09df7`),
`standard_uber_4lyr_pxm_hbb_fixuv1` (`b21f1ba5`) and
`standard_uber_4lyr_spx_dm_hbb` (`360e396b`) share **17 slots** (slot names
confirmed from recovered parameter names; data-map pairing measured: each
data map goes with one layer's diffuse in every material, 7/7 to 13/13):

| Slots | Meaning |
|---|---|
| p14, p12, p9, p13 | `diffusetexture_layer0..3` |
| p15, p10, p11, p7 | `bumptexture_layer0..3` |
| p4, p0, p5, p1 | unnamed data maps of layers 0..3 (BC7 or RGBA8: R ≈ 0, G about constant, **B a blobby height-like map**, A varies; `hbb` = height-based blend) |
| p2 | `lookuptexture` (weights) |
| p3 | `grassmasktexture` |
| p8, p16 | `tintbumptexture`, `tinttexture` (a generic tiling colour map, not a section map) |
| p6 | no texture bound |

A slot is a layer **index**, not a surface type: one texture can be layer 1
in one material and layer 3 in another.

- **Weights** (measured): the lookup is one BC7 2048² texture per 512 m
  section on TEXCOORD_0, which is exactly section position / 512 on all
  primitives. Alpha is 255; R + G + B ≤ 255 (99th percentile 258 from BC7
  rounding, median 229).
- **Channel to layer:** R, G, B → layers 0, 1, 2 (measured by a seam test
  over 19.6 K vertex pairs shared by two materials: the identity order
  scores 11,142 against 10,219 for the next; where a channel weight is
  above 0.5 its slot agrees across the seam 66 / 74 / 92 % of the time).
  **Layer 3 = 1 − (R + G + B)** (inferred, low on its own: the remainder is
  small and mostly at section borders, but the render check below supports
  it).
- **Reference:** a section's `_slod` tiles share one baked 1024² colour map
  (`terrain_lod_2lyr_grass_wetness`); a top-down render from the decoded
  fields matches its layout (section `s05_w19`).
- **Detail UV** (measured on samples): TEXCOORD_0 × 64 (one repeat per 8 m)
  by default. TEXCOORD_1 on `61e09df7` is world-planar at 8 m in places
  (e.g. u = x / 8 − 1151.5, v = −z / 8 − 255.5 on `s05_w18_001`) and
  unwrapped elsewhere; on `b21f1ba5` ("fixuv1") it is constant. TEXCOORD_2
  is a copy of TEXCOORD_0 or constant. COLOR0 RGB is 50–230 (not weights),
  A always 0.

**Per-layer constants** (measured; meanings unknown): vec4s in the shader's
constant block ([§7.6](07-models.md#76-shaders-materials)) whose component k
depends on layer k's diffuse texture (functional-dependency diagonal
0.83–1.00 over 1,638 materials), which independently confirms the layer
numbering:

| Hash | Float offset | Equal across a seam (same / different layer texture) | Values |
|---|---|---|---|
| `c35e27af` | 56 | 100 % / 44 % | 1.0–1.7 |
| `627ce8ae` | 40 | 99 % / 71 % | 1 or 2 |
| `781f7100` | 100 | 98 % / 10 % | 0.35–0.75 |
| `9a4a3781` | 88 | 92 % / 14 % | 0–0.175 |
| `f354f896` | 112 | 85 % / 13 % | 1–16 |
| `9ede7c13` | 124 | 83 % / 23 % | 1–14 |

A tiling test of `c35e27af` was undecided, and a brightness test against
the baked map had no power.

### Render check

(2026-11-23; from rendering exported minigame levels and comparing with
footage, so these are consumer-side confirmations, not shader decodes.
Layers are numbered 0–3 as above.) Shaders `61e09df7` and `b21f1ba5`:

| Item | Finding | Confidence |
|---|---|---|
| layers | four colour textures | high |
| weights | RGB of the lookup texture on TEXCOORD_0 give layers 0–2; layer 3 = 1 − R − G − B. A 4 × 4 lookup reading RGB = 0 (`3fb347dace547ef9`) shows layer 3 only, as intended | high |
| tint | a tint texture on TEXCOORD_0 multiplied over the result; the exact maths is open (`colour × tint × 2` at partial strength looks right) | medium |
| height blend | each layer's data texture carries a height (blue channel); the higher layer wins at a boundary | medium |
| tiling on 512 m map sections | TEXCOORD_0 × 64 = one repeat per 8 m | high |
| set pieces | the detail UV varies per material (section-tiled, TEXCOORD_1, TEXCOORD_2 or none); road lookups are tileable and run TEXCOORD_0 to about 13.8, so they need repeat wrapping | high |

Counts: a mountain-valley level of the minigame pack has 169 tiles and
1,675 four-layer materials, covering 6,022 of its 6,023 ground meshes;
three set-piece levels have 228, 45 and 10 (one of them 907 meshes: 697
section-tiled, 207 with no detail UV, 3 on TEXCOORD_1).

**Missing layer texture** (high): on tiles `s05_w18`, `s05_w19` and
`s05_w20` of the minigame pack, 24 four-layer materials (126 meshes)
reference one layer texture that is in no texture store on disk; the other
three layers, the weights and the tint are present.

Open: per-layer constant `c35e27af` (1.0 on 108 of 137 layers here; tiling or
intensity), the data-map channels, what COLOR0 RGB does, the tint maths, and the projection when a material has no
detail UV.

## Two-layer rock shader `prop_2lyr_batch`

(2026-11-23; render check.) Shader `89c8fe07`: base atlas on TEXCOORD_0, a
tiling second layer on TEXCOORD_1, **weight = 1 − COLOR0.a** (unpainted
vertices, A = 255, show the base only; with the weight not inverted every
unpainted rock is fully mossed). High. One level places 15,656 rock meshes
from 32 models and 41 materials; mean COLOR0.a is 0.63. A dirt overlay
uses TEXCOORD_2, which on rocks is parked at a constant, so the dirt is
uniform (low–medium).

**Normal maps are DirectX convention (green = +Y down)** (confirmed by an
integrability test on 270 of 300 maps). Invert G for glTF/OpenGL.

## Vertex colour red and green

Sampled on 4,000 models. Both channels are still **unknown**; every test so
far was negative (the export does not use them):

- **Red** looks like a baked shading term. It is mostly 255, about 23 on
  average on trees, and shows soft dark blobs on ground tiles where trees
  stand outside the model. It is about 252 on interior shaders, so it is
  not sky occlusion alone. Ray-cast sky visibility does not explain it,
  either from the model's own geometry or from the placed world around it:
  on two ground tiles cast against every placed model within 120 m, the
  correlation with red is about 0 (−0.02 and 0.06), and with green 0.06
  and 0.14 (confirmed negative result). Nearby instanced trees, the AO
  byte of the grass batches and the distance to the road do not explain it
  either.
- **Green** equals red on 63 % of ordinary vertices. On ground shaders it
  is low and changes from surface to surface (on one street tile grass 0,
  road mid grey, pavement 255), but it also varies widely within one
  collision surface (standard deviation about 54 of 255 on a road), so it
  is not a surface-type mask.

The numbers behind these results (measured on samples):

| Test | Result |
|---|---|
| mean R / G by shader group | ordinary 229 / 180 (G = R on 63 %), ground and terrain 213 / 57 (G = R 12 %), trees and foliage 23 / 126 (G > R 88 %), interior 252 / 125, emissive 219 / 178 |
| per shader | `standard_fabric` 255 / 254, `standard_2lyr_ground` 235 / 16, `trees_standard_wind` 40 / 116, `standard_anisotropy_dirt` 192 / 70 |
| own-geometry sky visibility vs R | median correlation 0.07 over 25 models (one panel model 0.79) |
| surface id (two HD ground tiles, up-facing vertices over their collision triangle, median gap 4 / 13 mm) | G: eta² 0.04 / 0.10 (road mean G 55, sd 54); R: eta² 0.19 / 0.08 (grass 202–216, road 252) |
| near instanced trees | mean R 248 within 2 m, 254 beyond 10 m; corr(R, trees within 10 m) −0.15 / −0.14 |
| grass AO byte, road distance | corr 0.01; −0.22 / 0.14 |

## Material presets (peds)

About 1,500 of the 4,678 meta resources in `packs.rpf` hold material
presets and sets (confirmed layout; field names are hashes):

| Struct | Fields |
|---|---|
| `f795e8ba` preset | name, parent, Category, a list of `37e52277` parameters, and two maps ([§8](08-meta.md), type 0x71): `12f85293` tags (key → u32 list) and `9a289423` |
| `37e52277` parameter | `3eaddcd7` (`StateName`) parameter hash (`44e21c90` = joaat("base")), a colour, `147fdab5` = four u64 texture hashes (`ttpf` keys, [§5](05-ttpf.md)) |
| `a9d57751` material set | name, a list of (slot, preset) pairs; face sets hold four skin variants in the "base" slot |

Tags in `12f85293` are joaat of the lower-case tag name: `002f77d9` =
"pedappearance", `a9473ab4` = "style". Key **`4bfd27dd` lists the drawables
the preset applies to** (joaat of the full drawable name); this is how a
head gets its face ([§7.5](07-models.md#ped-heads-face-textures-confirmed)).
Set and preset names do not resolve.

**Records are linked by u32 id, not by pointer** (2026-11-23; medium-high),
so a decode that follows pointers from the root misses them; dump every
struct instance instead. In the ped wardrobe resources (e.g. the zombie
ones, `packs.rpf@2b21010` female and `@2b34010` male): `a9d57751` is a
variation (`65121749` id, `Scale`, an `fb5bb20d` list of {`65121749` id,
`StateName`} links); `f795e8ba` an item record (tag map `12f85293` {Key,
Item[u32]}, Category, `080eec39`, `e13ebaf8`, an `ebacf18c` list of
`37e52277`); `37e52277` a state / texture set (`147fdab5` with 64-bit
texture hashes: normal, detail …). In these two files key `4bfd27dd`
always names a head (19 and 30 items, all resolving to `head_*` drawables;
high).

**How wardrobe items link to drawables** (2026-11-23; corrects the reading
above that `4bfd27dd` is the only drawable key):

- The `12f85293` map is a **tag map**; keys are tag types and values are
  joaat of a word or of a drawable-name prefix (`tanktopcasual_fr1_028` for
  `tanktopcasual_fr1_028_disheveled_un_full`). `4bfd27dd` head drawable,
  `023e5a2f` hair / brows / body hair, `2a8cdb1d` garment name or a shape
  word (high for these three); `a9473ab4` style (casual, sports, swim …),
  `e60617ae` colour, `fe330559` length, `928c398d` front / back, `3a57a0fc` (`pedtype`)
  body type (medium, named from the words they hold).
- A colour variant's `080eec39` (sometimes `e13ebaf8`) is its parent / base
  item id; Category is the slot (eyes, hair, hats, ArmorTorso, Helmets,
  Gilets, teeth …) (high).
- **Drawable link:** the 64-bit texture hashes in an item's `37e52277` sets
  (its own and its parents') are its drawable's material textures, so the
  model sharing the most of them is the drawable (6 of 6 on
  `tanktopcasual_fr1_028`; 84 of 118 zombie variations resolve). High for
  garments, lower for eyes and teeth, which share textures across body
  types.
- Each variation is a clothing set or a head set (head, eyes, teeth,
  eyecap).
- **Variation lists** (2026-11-23; medium): a separate packs PSO maps each
  model to its variations. The map key is the model hash; each item is a
  struct `a5ea6310` {`fc7eaa76` variation id, `fee182a6` outfit type,
  `Scale`}. Only clothing sets are listed; head sets are not, which
  suggests any head set can pair with any listed clothing set (inferred).
  Outfit types are shared between models. *Later (2026-11-23, confirmed:
  ids resolve):* map struct `78262a67`, key joaat(model name); items list
  the model's outfits in order with a fourth field, `548de055` (bool);
  `Scale` is `3cba3fe6`. 2,717 models, 33,688 entries; for 2,689 models all
  variations are in exactly one wardrobe resource. No default-outfit marker
  was found; the 6,275 outfit type values are unresolved.
- **Variations include variations** (confirmed on the resources checked): a
  variation's item list (`fb5bb20d`) also holds ids that are not items of
  the resource but other variations: a head set, a body set with the skin
  meshes, brows, a body-shape preset (tagged `BodyShapePreset`). A full look
  is the outfit plus everything it includes, recursively. *This weakens the
  "head sets pair freely" reading above: an outfit can name its own head
  set (not reconciled).*
- **Item states** (`37e52277`): `StateName`; `46d26d93` a flat ARGB colour
  (on about 14 K of 80 K garment items used by variations); and `e7cb9e1b`
  with `41acf8eb` asset GUID, `ead733fc` colour texture and `d689b17f`
  normal. For hair, brows, beards and lashes the GUID is the strand-hair
  asset (confirmed: 668 of 670 are in the piece table). A head item's state
  names the face albedo and normal (both in the texture stores); a body
  item's colour texture is in no store (only masks, detail and normal maps
  are), so body skin is probably built at run time (inferred). 54 % of
  garment items used by variations store neither a state colour nor tint
  bytes.
- **Colour tag** `e60617ae` (confirmed): 20 distinct values over 103,231
  tagged items; 18 are joaat of a colour word in lower case, capitalised or
  upper case (black, blue, white, brown, gold, grey, khaki, red, silver,
  navy, pink, green, purple, olive, yellow, orange, cream, beige), 99.5 % of
  uses; `f90074d1` and `b64fdfce` are unresolved.
- **Head tag `4bfd27dd` finds a wardrobe** when the model is listed under
  another name: 159 of 327 character prefixes resolve only this way
  (confirmed by count).
- **Shoes:** the name token before `_full` is a heel class: `fh` (178
  drawables, flat by geometry), `hh` 19, `sh` 7, `mh` 5 (inferred from names
  and geometry); the foot pose per class is not found.
- **Attached accessories:** a watch's mesh sits at its own root bone
  (`Watch_Root`, under `SKEL_ROOT` at the authoring origin), with unused
  attach bones `Watch_L_Wrist` / `Watch_R_Wrist` under `RB_L_Wrist01` /
  `RB_R_Wrist01`; moving the root onto an attach bone places it (one watch,
  by render). Item states named left / right probably pick the side (not
  checked).

**Dirt presets** (2026-11-23; high for the layout): plain XML in
`common.rpf`. `CannedColors` maps a key to an RGB colour; `AlphaChannels`
lists (Probability, Channel) pairs; `DirtDefinitions` hold scalars,
`ClothHideDirtModifiers`, `SkinDirtModifiers`, `AlphaMasksLookup`,
`ColorLookup` and `IgnoreZoneColor`. Which mask channel the game samples is
unknown.

**Clothing colour: wardrobe tint palettes** (2026-11-23):

- Ped clothing shaders (e.g. `ped_standard_muv`, `d5ed50f4`) bind no
  colour texture. The packed data map (`739e3d0d`) reads R = 255
  (251–255), G = fabric shade (about 60–205), B = flat per-region levels
  (0 to about 100), A about 127 (measured).
- **Both packed maps** (2026-11-23; channels measured, roles confirmed by
  render). *Correction to the bullet above:* the clean fabric shade is
  `speculartex` G, not `speculartex2` G; shading with `speculartex2` G
  mottles every garment. Each garment binds:
  - `speculartex` (`c6f46e79`, 512² or 1024² BC7, in the garment's UV
    layout): G flat mid grey with seams, B mid grey with fine fabric
    detail, R a dark noisy mask, A white.
  - `speculartex2` (`739e3d0d`, 256² BC7, also in the garment's layout, not
    tiling): R 255, G a blotchy pattern, B two flat levels (about 98 inside
    the garment, 63 outside), A about 128.
  - In the pixel shader (pass `PS_DeferredPedTexturedNC_28d6614a`; read
    from machine code, see the weapon shader in
    [§7.8](07-models.md#78-weapons-modular) for the same scheme):
    `speculartex` G is overlaid on the base colour and B added as linear
    light, both neutral at 0.5; R masks a second colour set.
    `speculartex2` never enters the colour: G modulates gloss and R is the
    amount of a wear overlay. The base colour is a three-stop gradient per
    sub-material over a tiling fabric texture, all held in a run-time
    parameter block whose source was not found.
- **Palette textures** (high): 21 textures in one `textures.rpf`
  dictionary (the `distant_window_emissives_atlas` txd):
  `ped_tint_generic_{clean,desaturated,vibrant}`,
  `ped_tint_garment_{clean,desaturated,vibrant}`, `ped_tint_hair`,
  `ped_tint_vstation_{generic,generic_desat,special,special_desat,master1–4}`,
  `animal_tint_{generic,generic_1,vibrant,vibrant_1,desaturated}` and
  `proxytint_palette`. Each is 64 × 256 R8G8B8A8_UNORM_SRGB, one mip, in the
  Gfx10 `SW_4KB_S` layout ([§4](04-xbox-tiling.md); decoding it with the
  dictionary's 16 × 16-tile order scrambles the 4-byte texels). Each row is
  one colour as a dark-to-light ramp in texels 2–55; texels 0–1 are a
  constant (20, 20, 20); texels 56–63 are one grey ramp, the same on every
  row.
- **Item fields** (high): in `f795e8ba`, `5ecf9331` = joaat of the palette
  texture name, and `c4d75d79`, `f4fb3dec`, `ab5d2a85` (u8, in schema
  order) are the palette rows of tints 0, 1 and 2; 255 = off.
- **Resolution rule** (medium, statistical): walk the item, then its
  `080eec39` parents; the first item whose three bytes are neither
  255, 255, 255 (inherit) nor 0, 0, 0 (unset) sets the tint. The palette is
  that item's own, or `ped_tint_vstation_generic` (`05317603`) when it is 0.
  Check against the colour tag (`e60617ae`) on 48,817 items tagged black,
  white, blue, red or green: 39 % resolve a tint; of those, tint 0's row at
  mid shade matches the tag 61 % of the time (`vstation_generic` 68 %,
  `vstation_generic_desat` 67 %, `vstation_special` 71 %, `generic_clean`
  27 %). As the default palette, `vstation_generic` matches 61 %,
  `vstation_generic_desat` 58 %, `generic_clean` 37 %. Byte 0 as the primary
  tint beats bytes 1 and 2 (15,486 against 12,233 / 12,320 matches).
  Palette 0 with tints 0, 0, 0 occurs on 48 K items, and `generic_clean`
  with 0, 0, 0 carries every colour tag, so that combination reads as unset
  or set at run time.
- Tint 0 probably follows the data map's G (medium); how R and B bring in
  tints 1 and 2 is unknown.
- Open: palette `fcef23e3` (eyes, teeth, belts) is not in that dictionary;
  the `e13ebaf8` link targets resolve to no item id or known name
  (*answered 2026-11-23: `e13ebaf8` is `GarmentPreset`, a key of the
  [layered material library](#layered-material-library)*); on a few
  sets `37e52277.46d26d93` holds ARGB colours (alpha `ff`), meaning
  unknown.

Open: whether head sets really pair freely with clothing sets, hair palette
`fcef23e3` (not `ped_tint_hair`), tints 1 and 2, which cut skin mesh goes
with which garment, body-shape preset values, the `StateName`
states, and most scalar fields of `f795e8ba`; the same structs appear in
hundreds of other resources, keyed by shared Category ids.

**Wardrobe field names** (2026-11-23; case-sensitive joaat, every name
re-hashed and matching; high for one- and two-word hits that fit the data):

| Struct | Fields |
|---|---|
| `f795e8ba` kit (the item / preset above) | `KitName` `65121749`, `TemplateName` `080eec39`, `States` `ebacf18c`, `Palette` `5ecf9331`, `Tint0` / `Tint1` / `Tint2` (`c4d75d79`, `f4fb3dec`, `ab5d2a85`), `GarmentPreset` `e13ebaf8`, `PropertyTags` `12f85293` |
| `37e52277` kit state | `StateName` `3eaddcd7`, `Asset` `e7cb9e1b`, `KitLayer` `1d52bc91`, `ToStateNames` `83698d93`, `PropertyTags` `12f85293` (its own tags, 2026-11-24) |
| `147fdab5` asset | `Drawable` `41acf8eb`, `Albedo` `ead733fc`, `Normal` `d689b17f` (2026-11-24), `Material` `9d1000d8` |
| `a9d57751` kit set | `IncludedKits` `fb5bb20d` |
| other | `AssetName` `7deba6f8` |

Still unnamed: `46d26d93`, `a41d353b`, `9a289423`, `35ff19d7`, `3b7601de`,
`7e43b9a9`, `b70a22c1`, `ba2e6e5b`.

Property tag keys (lower-case joaat; medium-high where the values fit):
`kittype` `2d497cdb`, `pedtype` `3a57a0fc`, `hairlength` `d35f809f` (values
`l01`–`l09`), `slodstyle` `2a8cdb1d`, `ethnicity` `6ba0d406`,
`dirtdefinition` `83cc4207`, `bodyshapeprimary` `36313cc7`, `goretype`
`ccd28d61`, `decorationpresets` `d9f1087b`, `weatheringcategories`
`33045a8d`, `outfitpedtype` `fa177698`, `noasset` `ca0f75a6`,
`audio_lower` `1630a5bc`, `audio_accessories` `32d6cdc7`, `audio_hands`
`37f749b3`, `audio_head` `77e42c9f`, `audio_face` `90705e5e`, `audio_shoe`
`b860b6b5`, `audiofabric` `ac83b156`. Hair style values include `bob`,
`lob`, `bun`, `topknot` and `cornrows`. *`2a8cdb1d` was read above as a
garment name or shape; the name `slodstyle` suggests a low-detail style
(not reconciled).*

**Item states in detail** (2026-11-24, from one ped wardrobe resource; fields
and names high, names re-hashed with joaat; readings marked inferred):

- Besides the fields above, each `States` entry has two mask lists:
  `35ff19d7` = list of {Key: layer hash, Item: u64 bit mask}, and
  `3b7601de` = list of {Key: layer hash, Item: list of {Key: 1 or 2,
  Item: u64 mask}}. The masks probably address the `GM_U01_*` region bones
  ([§7](07-models.md)) of the keyed layer (inferred); what keys 1 and 2
  mean is unknown.
- **State names:** `base` `44e21c90`, `hairhidden` `612d31ea`,
  `hathairalt` `93be327e`, `maskhairalt` `54685207`, `hatcapbackwards`
  `1465a09b`, `hatunderhood` `e8cddf6b`, `left` `6fa34840`, `right`
  `b8ccc339`, `raised` `ebfd59b5`. **Layer names:** `layer_1` `b4d9d470`,
  `layer_2` `e0622b84`, `layer_3` `d23c8f39`; layer `275822c4` is unnamed.
- **Tag values** resolved: `has_hair`, `has_headwear`, `glasses_down`,
  `glasses_raised`, `hood_down`, `covered_neck`, `covered_back`, `open`,
  `half_open`; tag keys `kittype`, `hairlength` and `style` (`a9473ab4`).
  Category names are plural words: `shirts`, `hoodies`, `pants`, `shorts`,
  `gloves`, `jackets`, `piercings`, `hair`.
- **Counts** in that resource: 271 items, 111 distinct state names (about
  100 still unnamed). 7 hair items have more than one state; all 7 have
  `base` and `hairhidden`, and `hairhidden` names the same asset GUID on
  every one. One hair item has 7 states, each alternate naming a different
  hair asset.
- Garment drawables carry one bone per body region, named
  `GM_U01_<side>_<region>_<part>` (69 on one upper garment).
- Reading (inferred): a hat switches the hair item to `hathairalt` or
  `hairhidden`. Open: the rule that picks a hair state for a given hat.

**Hair colour on items** (2026-11-23, counts high): of the hair items used
by outfit variations, about 14 K have tint bytes 0, 0, 0 (palette
`ped_tint_generic_clean` or none), about 330 name `ped_tint_hair`
(`1e64f584`) with a real row (rows 3–86 seen), which is the stored hair
colour of those characters, and 22 name `fcef23e3`, which is not one of
the 21 named palettes of the ped tint dictionary. The game probably picks
the hair colour at run time for the rest (inferred). *A second survey
(medium, absence of evidence) found hair kits with `Palette` = the general
clothing palette or 0 and `Tint0`–`2` always 0 or 255; this disagrees with
the 330 `ped_tint_hair` rows above and is kept alongside them.* A
re-check found that the hair items naming `ped_tint_hair` carry a real row
in `Tint0`, and the rest have 0, 0, 0 or 255, 255, 255, so the second survey
probably saw only the other palettes (inferred). The strand
hair colour is also not in the kit property tags or in the groom resource
header (512 bytes: bounds, counts, half-float shape parameters, a 16-byte
profile curve, no colours). The hair pixel shader reads its two ramp row
numbers from a 336-byte per-instance record built at run time
([shader notes](../structure/06-models-shaders.md#archive-layout-in-detail-2026-11-23)).

**Cut skin meshes** (2026-11-23, confirmed by name listing): besides the
whole meshes, the packs hold only `hand_<fit>_000_{croptop_a00, tshirt_a02,
tshirt_b00, tshirt_c05}` and `feet_<fit>_000_{pantshigh_a02 … a04,
shortshigh_a02, shortsmid_a02, pantsrip_a00, gstring_a00}`; female fits have
`_full`, male fits only `nude_cull`. Which garment selects which cut is
still unknown.

**Negative results** (2026-11-23): head-shape distance does not identify a
character (the nearest of 142 named female heads to `cs_lucia`'s is
`player_story000_00` at 2.0 mm against a median of 2.9 mm, but unrelated
pairs reach the same ratio); and the hair placeholder tag `023e5a2f` cannot
link a headless character to its model, because generic peds borrow named
characters' placeholders as stand-ins.

### Layered material library

(2026-11-23; structure high, read from decoded PSO data.) The per-material
parameter block that the clothing shaders (and, by the same scheme, the
weapon shader, [§7.8](07-models.md#78-weapons-modular)) read is filled from
a library of layered materials. Two PSO files in the game-data archive
share root struct `8419ab01`, which has four hash maps keyed by name hash:

| Map | Record | Large file | Small file |
|---|---|---|---|
| `42fa4d39` `TilePresets` | tile preset `6cc4c278` (first called "base material") | 7,994 | 556 |
| `fd540657` | layered preset `756aef9c` | 13,071 | 2,635 |
| `951cf6e3` `Tiles` | tile `83021d7a` | 432 | 129 |
| `2d71e4f7` `TransmissionProfiles` | transmission profile `440c9ecb` (has fields `scattering` and `Absorption`) | 184 | 7 |

Field names are case-sensitive joaat (2026-11-23, high: 13 hits forming
one consistent set, each re-hashed); the map `fd540657`, the sub-material
field `aadeb53e` and `c897af02` are still unnamed.

- A **tile preset** names one tile (`TileName` `03e52760`; all resolve) and
  optionally a transmission profile (`TransmissionProfileName`
  `04be3aea`). A **tile** holds the tiling surface textures
  `tilematerialmap` and `tilenormalmap`. All 125,640 sub-materials (15,705
  presets × 8) resolve a tile.

- Every record has four parameter lists: `FloatParameters` `1652fce8`
  (items `38f05860`), `TextureNameParameters` `0f208402` (items
  `f1cb6289`, 64-bit texture hash), `ColorParameters` `fc24d9c5` (items `e9d99e0a`, u32 colour; first read as A,
  R, G, B from high to low; *corrected 2026-11-23, high:* **A, B, G, R
  from high to low, red in the low byte**, checked against wardrobe colour
  tags: 83 of 85 items tagged blue, navy, brown, red, orange or yellow
  agree) and
  `UIntParameters` `9f6b5b03` (items `4b2625bb`). Item fields: `Name` (parameter name
  hash), `Index` (array element 0–7, 255 = scalar), `Value`, and
  `aadeb53e` = sub-material 0–7 (255 = the whole record).
- A layered preset adds `TilePresetNames` `4a1ebc36`, 8 tile preset names
  (one per sub-material; all resolve in the same file), the per-sub-material
  overrides `TileNameOverrides` `4bee7340` and
  `TransmissionProfileOverrides` `cfc17384` (8 optional names each) and
  `c897af02`, a 64-bit id.
- The wardrobe kit field `GarmentPreset` (`e13ebaf8`) is a key of the
  layered preset map (every value checked resolves). About 38 % of the kits
  in the largest wardrobe file have one, and those rarely have a
  `Palette`; about 54 % of one main character's listed wardrobe items
  (1,292 of 2,393) carry one.
- The main colour parameters are `tiletint` `f574261b` (4 entries per
  sub-material) and `stitchtint` `1c85bdf6` (3 entries, at preset level).
  How they become the shader's three gradient stops is not covered here.
- **Parameter names** (2026-11-23; joaat of lower-case text, each
  re-hashed; 68 of 99 recovered, 38 listed here). High for two-word names
  and for three-word names inside a family:
  - tile: `tiletint` `f574261b`, `tilescale` `01ae45d3`, `tilerotation`
    `8bbce09b`, `tileoffset` `9dfe6abf`, `tilesheen` `71c79bb7`,
    `tilemetallic` `6c75b6b2`, `tileflags` `521acac3`, `tilematerialmap`
    `c380e141`, `tilenormalmap` `1b689ba7`;
  - tint mask: `tintmaskmap` `d2b598e5`, `tintmasktint` `587b7f6d`,
    `tintmaskopacity` `568fd269`, `tintmaskscale` `5bb20a63`,
    `tintmaskrotation` `3c72fd1c`, `tintmaskroughness` `0a1d5d6a`,
    `tintmaskmetallic` `349d273f`, `tintmaskflags` `59303da7`;
  - pattern: `tintablepatternmap` `f4f6bfcc`, `tintablepatterntint`
    `dc2f61da`, `tintablepatternscale` `79ffce9a`;
  - other: `stitchtint` `1c85bdf6`, `dirttint` `4a7be614`, `dirtcontrol`
    `b5b4d917`, `sheencolor` `ec2767ca`, `sheenblend` `08d15d02`,
    `anisointensity` `e208e0c6`, `specularfresnel` `27b9b2fa`, `windscale`
    `76259dac`, `alphathreshold` `c391fd51`, `emissivemultiplier`
    `5eebed48`, `engravingcolor` `df293ca1`, `engravingheight` `a0c31b78`.

  Medium (three-word names of a `…blendamount` family):
  `dirtmapblendamount` `83d9ae8b`, `heightmapblendamount` `14320490`,
  `roughnessmapblendamount` `cf86f980`, `detailnormalblendamount`
  `545a21f2`, `albedovariationblendamount` `8dffd2cd`,
  `curvatureblendamount` `58645355`.
- *Correction:* `d2b598e5` is not the tiling fabric texture but
  `tintmaskmap`, single-channel masks of extra tint layers (5 entries); the
  tiling textures are the tile's two maps.
- **Tint-mask layers** (2026-11-23; offsets and formula high, read from
  the clothing pixel shader; which named parameter feeds each field medium,
  matched by array shape and name). Each sub-material has four mask
  layers L = 0–3 in the 0xF00-byte parameter block (offsets per
  sub-material add sub × 16):

  | Block offset | Contents | Parameter |
  |---|---|---|
  | +0x810 + L × 4 | mask texture index; the red channel is the mask | `tintmaskmap`[L] |
  | +0x310 / +0x390 + L × 0x100 | 2 × 3 UV transform, applied to the same tiling UV as the tile texture. The tiling UV is the garment's **second UV set**, laid out at one scale on every mesh of a garment (2026-11-24) | `tintmaskscale`[2L, 2L+1], `tintmaskrotation`[L], `tintmasktranslation`[2L, 2L+1] (`b3643800`) |
  | +0x31c + L × 0x100 | flags; bit 2 inverts the mask (values seen 8, 0, 12 and rarer ones; the other bits are unknown) | `tintmaskflags`[L] |
  | +0x39c + L × 0x100 | opacity | `tintmaskopacity`[L] |
  | +0xa10 + L × 0x80 | three RGBA-byte gradient stops; second colour set at +0xc10 + L × 0x80 | `tintmasktint`[L], [4 + L] |
  | +0x710 + L × 4 | emissive intensity | `tintmaskemissiveintensity`[L] (`6f235650`) |
  | +0x890 + L × 4 | packed metallic, roughness and their opacities | – |

  Composition, base first, then L = 0, 1, 2, 3 in order, with g the tile's
  grey value:

  ```text
  colour = gradient(tiletint, g) * a + (1 - a) * g     # a = tiletint alpha
  per layer: t = mask * opacity * tint alpha
             colour = lerp(colour, gradient(tintmasktint[L], g), t)
  ```

  2,529 of 13,071 clothing presets set an active layer themselves, and
  5,943 of 7,994 tile presets carry mask textures. Open: the other flag
  bits, the fifth `tintmaskmap` entry, and how the pattern, stitch, dirt
  and sheen layers are combined.
- **Later layers** (2026-11-23; medium, read once from the clothing pixel
  shader):
  - **Tintable pattern:** the one-texture form of the four mask layers.
    One four-channel texture (`tintablepatternmap`, on the tile UV) gives
    R, G, B and A as the four masks, and the layer colours come from
    `tintablepatterntint`[0–5] instead of `tintmasktint`.
  - **Stitches:** a four-channel texture on a third UV input, with G as its
    grey value. The three `stitchtint` colours (+0xec0, +0xed0, +0xee0)
    each run a gradient over that grey: the first is the base, the second
    is blended in by the texture's R and the third by its B (each times the
    tint's alpha); the result replaces the colour by a weight not yet
    traced. Which texture this is was not identified.
  - **Dirt:** weight = `dirtmapblendamount` (+0xe30) × the second packed
    garment map's R; above a threshold the colour is pulled towards a
    two-colour ramp from `dirttint`[0] and [1] (+0xe34, +0xe38), with
    `dirtcontrol` at +0xe3c.
  - The first packed garment map's G (overlay blend, pivot at +0xe18) and B
    (linear light, pivot at +0xe1c) belong to the albedo-variation
    parameters (`albedovariationblendamount` is the pair at +0xe20).
  - The parameter defaults are in none of the 22 indexed archives (raw scan
    for the parameter hashes, and for the name strings in the two main data
    archives); 30 of 99 parameter names are still unknown.
- **Tint bytes** (medium, data only): the wardrobe `Tint0`–`2` bytes do not
  choose between the two `tiletint` entries; over 4,607 colour-tagged items
  the tag fits entry 0 (mean RGB distance 71) better than entry 1 (109)
  whatever the bytes are.
- **Clothing sub-material per vertex** (2026-11-23; encoding medium-high,
  colours not compared against the game): a garment with no `materialid`
  texture numbers its sub-materials in the vertex colour stream, green
  byte = 4 × id + 2 (2, 6, 10 … 30), id 0–7 indexing the garment preset's
  per-sub-material colours; garments with a `materialid` texture use it
  instead (red × 63). Every garment inspected (a few dozen: boots,
  sneakers, jackets, T-shirts, jeans) held only such values, the same
  encoding as on weapon parts ([§7.8](07-models.md#78-weapons-modular)), and
  belts, soles and laces come out as separate colours. Open: the red, blue
  and alpha bytes of that stream on clothing.
- **Preset against tint row** (medium; nearest of 18 colour words, neutral
  tags excluded): over about 6,100 items with a colour tag, a garment
  preset and a tint row, the preset's sub-material 0 colour matches the tag
  on 2,970 and the tint row on 390; on the `vstation` palettes it is closer
  (tint row 327, preset 284). What tint rows do on items that also have a
  preset is unknown.
- **The small file is the weapon library** (high). In the 65 weapon family
  PSO files, the per-component asset struct `85492dc3` has field
  `72f5e33e` = a layered preset key (1,005 of 1,006 non-zero values
  resolve), and struct `6724161c` {`Name`, `c763edb4`, `14b064c2`} has
  `14b064c2` = a layered preset key (1,300 of 1,301), a named variant list
  per family. `85492dc3`.`873ef899` is joaat of the component's model name
  (972 of 1,006 records resolve, 894 distinct models, 57 with two presets);
  `6724161c`.`c763edb4` is the part the variant recolours (frame, body and
  other part-name hashes).
- A second wardrobe field also holds clothing-library preset keys:
  `garmentPreset` `e830a068` of struct `12712917` (every value resolves);
  its role is unknown.
- Open: 30 parameter names; where the parameter defaults come from (not in
  any indexed archive); the source of the per-part blend byte; the role of the kit's
  `Tint0`–`2` on preset kits; the stitch weight and texture; the sheen
  layer.

### Ped model table

(2026-11-23.) One PSO in the packs holds 2,714 records of struct
`eb66d086` (`CPedModelInfo__InitData`), one per ped model, `Name` = joaat of the model name.

| Field | Contents | Confidence |
|---|---|---|
| `dc3dfe39` | a character name hash: 1,505 distinct values; for 47 of 114 `cs_` models it is joaat of the model name without `cs_`; zero on 35. `player_story000_00` holds joaat(`lucia`) and `player_story000_01` joaat(`jason`), each on exactly one record | confirmed for those two and `cs_baelux`; real field name unknown |
| `57134685`, `7689b960` | a gender-class name (`GEN_FEMALE`, `GEN_MALE` …; 5 distinct values) | high |
| `877619b0`, `df0e1fb5` | 159 and 49 distinct values, unresolved | – |

This is the only stored link found between the cutscene prefixes
`cs_lucia` / `cs_jason` and the protagonist models.

### Animal shader `ped_standard_animal`

(2026-11-23.) On shader `295b85a9` the colour slot can hold a **region
mask instead of colour** (high): on the white-tailed deer
(`crm_deer_whitetailed_m_00` / `_f_00`) the BC1 texture is linear data with
G = coat, B = white parts (belly, tail, throat, eye ring), R = dark parts
(nose, hooves, eyes). 78 of the shader's 1,384 materials use such a mask;
997 bind a real sRGB colour texture (the bald eagle among them). The coat
colours are not stored with the model: material constants from offset 64
(including a 16 × vec4 block `f08653a5`) and texture slots 2 and 3 are
filled at runtime. Per-model variation lists exist (buck 14 entries, doe 8,
eagle 4) but are unnamed hashes. Open: where the coat colours come from and
which variation a spawn uses.

### Tint palettes

Slot `2c8d7182` (`tintpalettetex`, table above) is a palette embedded in the
model. **Vertex colour blue (`COLOR0.b`, stream 1) indexes it** (confirmed:
on 36 sampled models the index never reaches a filler `ffbbff` entry).
Tinted parts have bright, near-grey base textures (mean saturation 0.05),
so the tint multiplies the base colour (inferred): white pool tiles come out
pool-blue, white cables black, grey or blue.

- **Layout** (confirmed): a palette is 2^n + k texels wide. Each row holds
  256 >> k entries (at most 2^n), and there are 2^n / row length rows; rows
  are colour variants. Widths seen: 13, 20, 37, 66, 69, 129, 131, 256, 260,
  513, 517, 1,024, 2,051 and 8,194. For example 1,024 = 2^10 + 0 gives 4
  rows of 256 (rows start at 256 / 512 / 768), and 131 = 2^7 + 3 gives 4
  rows of 32; also 260 = 16 × 16, 1,028 = 64 × 16, 8,194 = 128 × 64 and
  13 = 1 × 8. Later rows start with white; unused rows are white, then
  filler (e.g. `prop_am_shelf_metal_06b` 4 schemes,
  `prop_am_cabinet_filing_01b` 16). The palette key is
  *(texture struct +0x30) + 8; palettes are stored linear (no tiling).
- **The entity's `tintValue`** (`3c852527`, [§9](09-placements.md)) selects
  the row (confirmed: for every palette width, the largest `tintValue`
  placed with it is rows − 1). Row 0 is the default.
- Colour = palette[row × row length + `COLOR0.b`].
- **Counts** (all `levels.rpf` placements, measured): 362 K of the 1.63 M
  entities on palette models use a row above 0. 904 (0.25 %) exceed the
  row count, nearly all on 13-wide single-row palettes, so assume clamping
  (inferred). *This disagrees with the "largest `tintValue` = rows − 1"
  check above; both are kept until re-measured.* 120 K entities with
  `tintValue` > 0 are on models without an embedded palette (values mostly
  up to 60), probably another use of the field (unknown).

**Render check (2026-11-22).** The rule above, multiplied into the base
colour and applied only to primitives whose own shader binds
`tintpalettetex`, was rendered in a 3D engine:

| Finding | Confidence |
|---|---|
| **Row 0 looks plausible.** A pier Ferris wheel hub and plaza come out white / grey, with coloured entries (green, cyan, red at indices 120–125) only on small parts. There is no in-game reference image | medium |
| **Rows above 0 look wrong as read.** Rocks on a 260-wide palette (16 rows × 16) placed with rows 2, 6 and 7 render near black; on the hub's 1,024-wide palette (4 × 256), rows 1–3 are almost all `ffbbff` filler. So the row rule is probably **incomplete**: either `tintValue` does not select the row directly (the clamp at rows − 1 is inferred), or some shaders choose the row differently. Treating the palette as sRGB instead of linear does not fix it | medium |
| **LOD drawables often bind no palette** even when the full model does, and there `COLOR0.b` is not a tint index (the Ferris wheel LOD has b = 0 with about 100 other distinct values) | high |

This contradicts the "confirmed" row selection above for rendering; the
`tintValue` maximum per width still holds, so both are kept until the row
lookup is understood.

**Texture load of map pieces** (measured on a full map export,
2026-11-22; high): a 64 m HD ground tile such as `n02_e01_002_hd` references
71 materials and about 170 textures, about 166 megapixels; big ground tiles
and building shells typically use 120–170 textures each.

**Texture hashes in no store.** Embedded tint palettes account for 30,095 of
the 33,770 texture hashes that a full `levels.rpf` map export finds in no
`data/ttdb` store. The other 3,675 are in no store or resource of the
install, the content-pack stores included ([§5](05-ttpf.md); they serve the
packs' own models). They make up about 139 K references in a full map
export; `1e2c9340e14a7d07` and `aa3eeff23a15dc75` are probably
engine defaults (inferred). A survey of 3,375 models (2026-11-23) finds 715
base-colour slots over 79 further unresolved hashes: `a800afccf01aa36b` on
nearly every vehicle's `standard_vehicle_emissive_layer` (321 slots; a
runtime screen overlay, [§11](11-vehicle-paint.md#vehicle-shaders-2026-11-22)),
`45fecd22c2d1ef69` on ped heads, `1e2c9340e14a7d07` on batch LODs and
terrain decals, `32de36bd78efc4e4` and `a8a97b37c9e0af76` on airport
buildings; probably more defaults or runtime-bound textures (not
verified).

Uses by slot (full `levels.rpf` export, palettes excluded; measured):
`c6f46e79` 41.9 K (22.7 K of them `aa3e…`), `c8e8c282` base colour 19.3 K
(10.2 K `1e2c…`), `2c8d7182` 16.7 K, `739e3d0d` 10.1 K, `1e77bf4c` normal
8.6 K, `2f4646cf` 7.0 K (all `1e2c…`), `38ebcdb3` 3.2 K, `8e0d63ee` 2.8 K,
`9c362b2c` 2.3 K. `1e2c…` is in base and colour slots of standard,
decal_blend, alpha, 2lyr, 2lyr_interior and uber_4lyr shaders; `aa3e…` in
`c6f46e79` and the normal-atlas slots `2bb2ede8` / `4d56d7fe`. Typical
decoded values of each slot (median mean RGBA over 3,000 models), usable as
fallbacks: `c6f46e79` 0 / 150 / 224, `739e3d0d` 0 / 0 / 0, `38ebcdb3`
85 / 84 / 91, `3c4af3d4` 0 / 0 / 123, `2bb2ede8` 0 / 172 / 128, `4d56d7fe`
0 / 176 / 129, `1e77bf4c` 127 / 127 / 255, `c8e8c282` 159 / 152 / 145.

## Open questions

- 152 of 229 parameter names (mostly constants), and the four per-layer
  slots of the ground shaders; the collision surface hash
  ([§13](13-collision.md)) is a different, unsolved one.
- The 64-bit texture hash ([§5](05-ttpf.md)): FNV-1/1a 64, xxHash64,
  MurmurHash64A, CRC-64 (ECMA, XZ, ISO) and a 64-bit one-at-a-time over
  46 K known names (texture-dictionary texture names, model names, XML
  identifiers), as given, lower and upper case, match no texture hash or
  either 32-bit half beyond chance.
- Whether `c6f46e79` G is roughness or gloss; emissive and glass flags for
  unknown shader families.
- How tint palette rows above 0 are selected (see the render check under
  [Tint palettes](#tint-palettes)).
- Emissive intensity: no brightness for lamps, neon or lit windows is
  decoded, so a port has to choose its own (the emissive texture is the
  base colour image; brightness is unknown).


---

← [11. Vehicle paint and vehicle tables](11-vehicle-paint.md) · [Index](README.md) · [13. Collision and surface categories](13-collision.md) →
