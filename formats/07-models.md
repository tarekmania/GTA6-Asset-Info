# 7. Drawables, fragments and vehicles

*Part of the [SixNG format specification](README.md). Numbers as of 2026-11-21; confidence: confirmed / inferred / unknown.*

Checked on a vehicle body and on several map buildings and road pieces:
exported GLBs opened in a glTF viewer (shaded, normals, UV checker,
textures, skeleton).

## 7.1 Buffer descriptor (system page)

| Off | Meaning | Confidence |
|-----|---------|-----------|
| 0x08 | u32 size in dwords | confirmed |
| 0x0C | u16 4 | unknown meaning |
| 0x0E | u16 8 or 9 | unknown meaning |
| 0x10 | flags: 0x1601 = mesh blob (vertices), 0x1380 = meshlet index buffer, 0x0982 = ? | confirmed (first two) |
| 0x18 | graphics pointer | confirmed |

## 7.2 Mesh blob (vertex buffer)

u32 magic `0x4042` at 0.

| Item | Meaning | Confidence |
|------|---------|-----------|
| stream table | u32 slot k at +4k, k = 0..22. Value = `offset_in_2_byte_units << 8 \| format`; byte offset = `(v >> 8) × 2`. Slot 0 `0x4042` → offset 0x80, format 0x42 | confirmed |
| 0x60 | f32 × 3 position base | confirmed |
| 0x6C | u32: the model axis held in the flat 16-bit z slot. 2 = z: slots hold (x, y, z), identity. 1 = y: slots hold (z, x, y). 0 = x: slots hold (z, y, x), a swap and not a rotation. Before 2026-11-20 mode 0 was read as (y, z, x), which tipped wall panels by 90°. SixNG also scores all six permutations against the stored normals; **trust +0x6C instead**: a survey of 3,375 models (2026-11-23) found that override firing on 114 of 36,305 meshes (0.31 %, 51 models), and in the 7 checked by hand the hint was right and the override wrong (confirmed for those 7, inferred for the rest). Supporting numbers (measured on samples): on meshes with 50 or more scored triangles the hinted order scores ≥ 0.95 against the normals on 90.5 %, and only one mesh (a helicopter rotor piece) has a hint below 0.7 with another order at ≥ 0.95; six of the seven re-exported with the hint have equal extents on every LOD. Fall back to scoring only for values other than 0–2. Whether the header is ever wrong is open. | confirmed (2026-11-20) |
| 0x70 | f32 × 3 position step | confirmed |
| vertex count | V = (smallest stream offset > 0x80 − 0x80) / 8 | confirmed |

**Vertex streams** (tightly packed arrays of V elements):

| Slot | Fmt | Size | Meaning | Confidence |
|------|-----|------|---------|-----------|
| 0 | 0x42 | 8 | position (below) | confirmed |
| 1 | 0x26 | 4 | RGBA8 vertex colour: **b = tint palette index** ([§12](12-materials.md#tint-palettes)); a = layer-2 weight on multilayer shaders; r and g unknown ([§12](12-materials.md#vertex-colour-red-and-green)) | confirmed (b, a); r, g unknown |
| 2 | 0x21 | 4 | normal, octahedral 10:10 | confirmed |
| 3 | 0x28 / 0x42 | 4 / 8 | UV0: 2 × s16 / 8192, or 2 × f32 | confirmed |
| 4 | 0x28 | 4 | UV1 | confirmed |
| 6 | 0x26 | 4 | skin weights, 4 × u8 (sum 255), with slot 8 format 0x25 | confirmed |
| 8 | 0x00 | **1 or 2** | skin joint: one bone index, weight 1. Width = section size / V (next section offset − this offset). 1 = u8, **2 = u16 LE** | confirmed |
| 8 | 0x25 | 4 | skin joints, 4 × u8 (wheels, tyres, ped clothing) | confirmed |
| 8 | 0x4A | 8 | skin joints, **4 × u16**, weights in slot 6 (2026-11-23). On the 284 `_hi` vehicles' LOD 0 meshes: 440 meshes in 159 models (10.4 M vertices: body, trim, door cards); indices stay below the bone count (max 383 of 384); 4 of the 440 have an index that does not resolve. Slot 8 elsewhere: format 0x00 4,644 meshes, 0x25 382, 0x8C 157, none 2,551 | high |
| 7 | 0x26 | 4 | ped meshes: a second set of 4 skin weights (the 8 weights sum to 255) | confirmed |
| 9 | 0x25 | 4 | ped meshes: a second set of 4 skin joints, with slot 7 | confirmed |
| 11 | 0x27 | 4 | tangent R10G10B10A2: xyz = v / 1023 × 2 − 1 (model space); A2 = 3 → bitangent sign w = −1. **The frame is the second UV set's** (2026-11-23, high on two models with two UV sets: a tank fragment agrees with UV1 on 100 % of its surface and with UV0 on about 50 %, i.e. chance; ped jeans 99.9 % with UV1). cross(n, t) × w runs along +v of UV1 (v down the image), and normal maps are DirectX style (green = +v). Open: the frame for normal maps sampled on UV0, and whether single-UV meshes store a UV0 frame | confirmed; frame high |
| 12, 13 | 0x26 | 4 | RGBA8. *Slot 12 on vehicle bodies (2026-11-24, high): the fourth byte is the vertex's **sticker zone** ([liveries](11-vehicle-paint.md#vehicle-liveries)); the other three bytes are not understood* | unknown; slot 12 byte 4 high |
| 14, 15 | 0x28 | 4 | UV2, UV3 | confirmed |
| 17 | 0x42 | 8 | ? | unknown |

**Position** (one u64 q per vertex; RAGE Z-up, metres):

```
x = ((((q >> 16) & 0xFFFF) << 8) | ( q       & 0xFF)) * step.x + base.x
y = ((((q >> 32) & 0xFFFF) << 8) | ((q >> 8) & 0xFF)) * step.y + base.y
z =  (q >> 48)                                       * step.z + base.z
```

Bits 0–15 hold the low byte of x (bits 0–7) and of y (bits 8–15), giving x and
y 24 bits of precision (confirmed: with them, decoded geometry reproduces the
stored per-geometry bounding boxes to about 1.5e-5 relative over 782
geometries; without them x and y are off by up to 255 steps).

**Axis rotation (confirmed):** the quantiser puts
the flattest model axis of each blob in the 16-bit z slot, and the header
u32 at +0x6C names that axis (corrected 2026-11-20):

| +0x6C | flat axis | slots (x, y, z) hold model axes | order |
|---|---|---|---|
| 2 | z | x, y, z | (0, 1, 2) |
| 1 | y | z, x, y | (2, 0, 1) |
| 0 | x | z, y, x (a swap, not a rotation) | (2, 1, 0) |

Mode 0 was earlier read as the cyclic (1, 2, 0). Where the stored normals
could not tell the two apart, wall panels and slabs came out tipped by 90°
(on one tower drawable, 25 of 51 geometries fell outside the drawable
bbox). To check a blob, score all six permutations by how well the face
normals agree with the stored normals (always model space) and keep the
header's order unless another wins by more than 0.05. On that tower the
header's order scores 0.91–1.0 on every geometry.

**Normal:** `x = (w & 1023) / 1023 × 2 − 1`, `y = ((w >> 10) & 1023) / 1023
× 2 − 1`, `z = 1 − |x| − |y|`; if z < 0 fold x and y (standard octahedral
unfold), then normalise. The top 12 bits are unknown.

**Float-position variant (confirmed):** cables and pool/fountain water.
Slot 0 is `0x3063` (format 0x63 = 3 × f32 at 0x60), with no base/step
header; V = (next stream offset − 0x60) / 12. Slot 2 (also 3 × f32) is the
normal on water, and the unit direction along the wire on cables. Cable
vertex pairs share a position (zero-width ribbon widened by the shader;
UV0.x = ±1 gives the side). SixNG widens cables into two crossed 2 cm
ribbons; the real thickness is a shader parameter not yet read. A ribbon
can only be widened this way when every position occurs exactly twice:
10 of 53 surveyed cable meshes break that rule, probably where two cables
share a position (inferred, not checked on raw data).

## 7.3 Meshlet index buffer (flags 0x1380, confirmed)

| Off | Meaning |
|-----|---------|
| 0x00 | u16 meshlet count n |
| 0x02 | u16 ? |
| 0x04 | **u32** triangle count t |
| 0x08 | **u32** remap count r |
| **0x0C** | n × 16-byte descriptors: u32 ?, **u16 remap start**, u16 triangle start, u32 ? (cone?), 2 × f16 ? |
| 0x0C + 16n | t × u32 triangles: local indices in bits 0–6, 7–13, 14–20; meshlet number from bit 21 |
| … + 4t | r × u16 remap (local → vertex) |

*Correction (2026-11-23):* the triangle and remap counts are u32; they
were first read as u16 followed by a zero u16, which drops triangles on big
meshes (on a motorcycle body: 595 meshlets, 70,057 triangles, 69,901 remap
entries, 65,534 vertices; 4,521 triangles were lost before). The u16 remap
start wraps past 65,535 but ascends, so unwrap it along meshlet order
(confirmed on that mesh). Over a survey of 3,375 models (all 305 vehicle
`_hi` bodies plus samples of parts, props, map, weapons, animals and peds)
the u16 reading broke 44 models, all vehicles (41 of the 305 bodies,
13.4 %): 27 shredded (triangle count ≥ 65,536) and 17 missing parts (remap
count ≥ 65,536; e.g. `ambulance_hi` geometry 19, 59,541 triangles and
67,575 remap entries, exported only 3,422 triangles and lost its interior).
With u32 counts nothing is shredded, and triangle totals rise 1.4× to 16×
(`rhino_hi` 53,808 → 301,345). **Rule of thumb:** 1–12 dropped (degenerate)
triangles on a mesh is normal; more than about 30 on a mesh of tens of
thousands of vertices means a decode problem.

Vertex = `remap[remap_start[meshlet] + local]`. Drop degenerate padding
triangles. Descriptors start at 0x0C, not 0x10. About 97 % of triangles
agree with the stored normals; flip the rest.

## 7.4 Geometry, model and LOD table

| Structure | Field | Confidence |
|---|---|---|
| geometry record (0xA0) | +0x08 ptr → mesh-blob descriptor; +0x40 ptr → holder whose first u64 points to the index-buffer descriptor | confirmed |
| model (stride 0x40) | +0x00 ptr → geometry-record ptrs; +0x08 u16 count, u16 capacity; +0x0C u16 (peds: bone-palette count); +0x10 ptr (bounds?); **+0x18 ptr → u16 shader index per geometry**; +0x20 ptr (peds: **bone palette**, [§7.5](#ped-skinning-bone-palette-confirmed)); on vehicles +0x0C / +0x20 hold something else (+0x0C = 295 on a car body, entries not sorted); +0x28 u32, u8 0xFF, u8, u16 count | confirmed (0x00, 0x08, 0x0C, 0x18, 0x20) |
| model-array header | 16 bytes {ptr → model-pointer array, u16 count, u16 capacity} | confirmed |
| LOD group | pointers to model-array headers at **+0x50, +0x58, +0x60, +0x68** of the main drawable (high, med, low, vlow; empty = 0). Main drawable = the root of a `.#dr`, the struct at root+0x18 of a `.#ft` | confirmed (2026-11-20) |
| embedded drawables | other model arrays belong to embedded drawables: fragment children (root+0x10, records 0xC0 apart, each with its own +0x50 list) and nested sub-drawables in buildings. They are parts, not LODs | confirmed layout; placement of fragment-child parts inferred |
| drawable dictionaries | a resource whose own root has no LOD table can hold **several member drawables** (most map LOD / SLOD models are stored this way). Each member root has its own LOD group at +0x50…+0x68, a pointer at **+0xA8 to its name** (`<name>.#dr`, NUL-terminated, in the system page) and its own shader array (member root +0x10 → +0x10); model shader indices are local to the member. Members with the same name are one member; a resource with one named drawable, a fragment, or a root with its own LOD table is not a dictionary (other named drawables there are embedded parts). With members linked by name, 99.98 % of map placements resolve to a model (2026-11-22) | confirmed |
| drawable bounds | root +0x30 / +0x40 bbox min / max (drawables only) | confirmed |
| fragment bounds | fragment root +0x18 → its drawable, whose +0x30 / +0x40 hold the bbox | confirmed |
| stored bbox reliability | not always the geometry's extent: on route-sign prefab fragments it is shifted about 4.3 m in x, and on one scaffolding model it is rotated 90° about z. The archetype definition's bbox matches the decoded geometry in both cases (as of 2026-11-20) | confirmed (2 cases) |

Each model struct is preceded by 8 bytes and referenced at struct − 8 from
an array of model pointers. On a vehicle body, LOD1 is the same parts with
about 3× fewer vertices (not a damage set). Ranking every model array by
file offset gives wrong LODs: on fragment props (streetlights, signs) a
base plate or bracket became LOD0 and the full pole LOD2, and on buildings
trim strips from sub-drawables became LOD1–3. Use only the LOD group. In a
sample, 8 % of drawables and 2 % of fragments have such parts (aircraft:
5–16, flat rotor-height pieces in model space). Map buildings usually have
a single LOD: use the placement LOD hierarchy ([§9.2](09-placements.md))
for distance LODs.

## 7.5 Skeleton (fragments, confirmed)

Bone record (0x50 bytes):

| Off | Meaning |
|-----|---------|
| 0x00 | f32 × 4 local rotation quaternion (x, y, z, w) |
| 0x10 | f32 × 3 local translation, then u32 hash |
| 0x20 | f32 × 3 local scale |
| 0x30 | ptr → name |
| 0x40 | u16 flags |
| 0x42 | u16 ? (3 or 7) |
| 0x44 | u16 next sibling (0xFFFF none) |
| 0x46 | u16 parent (0xFFFF root) |
| 0x48 | u16 index |
| 0x4A | u16 name hash |

Skeleton header, found by its pointer to the bone array:

| Off | Meaning | Confidence |
|-----|---------|-----------|
| +0x00 | ptr → bone array | confirmed |
| +0x08 | ptr → **inverse bind matrices** (4 × 4 f32 per bone; row-vector convention `p_bone = p_model · M`) | confirmed |
| +0x10 | ptr → local bind matrices (w lanes hold other data) | inferred |
| +0x18 | ptr → u16 parent indices | confirmed |
| +0x20, +0x28 | ptrs | unknown |

Inverting the inverse bind matrix gives each bone's model-space pivot.
Checked symmetric and in place on a car: wheels, bonnet and boot hinges,
door hinges, lamps. Vertices are in model space (bind pose). Most vehicle
parts use one bone per vertex; wheels and tyres up to 4.

**u16 bone indices (confirmed, found 2026-11-20).** Slot 8 keeps format
0x00 but is **2 bytes per vertex** on some geometries of skeletons with
more than 255 bones (a high-LOD car body has 295). The format code does not
say so; only the section width does. Reading it as u8 put interior, lamp,
window and bumper geometry on the wrong bones. SixNG writes `JOINTS_0` as
u16 when any index exceeds 255. **Vehicle GLBs exported before this fix
should be re-exported.**

Bone, joint and attach-point names (`suspension_lf`,
`bonnet_catch_l_pos1_attach` …) are plain strings in the system page.
`wheel_[lr][fmr]N` are the wheels. On a motorcycle `wheel_turn_lf` is the
steering / fork bone (high: rotating it moves the fork), and
`wheel_parent_lr` and `wheel_lr_drive` carry the swingarm and the drive
(medium). Open: why slot 8 uses format 0x4A instead of a width-8 format-0
stream, and whether the 2,551 vehicle meshes without slot 8 bind to a bone
some other way.

**Bone names are not only letters, digits and underscores** (2026-11-23,
high): vehicle skeletons hold breakable-joint bones named like
`connect-chassis-bumper_f-1`. A reader that ends the bone array at the
first such name cuts the skeleton short; on a motorhome it stopped at bone
107, before the wheel bones, leaving 18 meshes with joint indices past the
bones read. The array continues with consecutive index fields, and with
dashes allowed the wheel bones appear and the wheels mount. Allow at least
`-`, `.`, `#`, `+` and space; the full character set is unknown.

**More vehicle bones** (2026-11-23):

- **Dual wheels** (high): the inner wheel of a pair has its own bone, the
  outer wheel's name plus `_inr` (`wheel_lr_inr`, `wheel_lm1_inr`,
  `wheel_rf_inr`), and its own [wheel disc](13-collision.md#vehicle-fragment-bounds-and-wheel-discs).
  Outer and inner discs: x = 1.44 and 0.98 on a semi-trailer chassis
  (radius 0.6, half width 0.174), 1.11 and 0.99 on a small baggage trailer
  (radius 0.18, half width 0.052). Two equal discs can lie within 12 cm on
  one axle line, so match each wheel bone to the nearer disc.
- **Wheel helpers** (high for the names): `wheel_turn_<pos>` (steering),
  `wheel_<pos>_drive`, `wheel_parent_<pos>`, `hub_<pos>`,
  `spacer_wheel_<pos>`, `camber_<pos>`. A train car has `bogie_f` and
  `bogie_r`, with four wheel bones each (`wheel_lf`, `wheel_lm1`,
  `wheel_rf`, `wheel_rm1`; `wheel_lm2`, `wheel_lr`, `wheel_rm2`,
  `wheel_rr`). Rail wheels use `rim_train_001` with no tyre in the
  configuration; that rim reads as a wheel with an axle stub (bbox x −0.82
  to 0.39 at radius 0.5; medium).
- **Convertible soft top** (bones high): a bone `convertible_roof` with 39
  descendants on the one car looked at: `frame_base`, `top_frame_front` /
  `_rear`, `tension_bow`, `tension_frame_front` / `_rear_upper` /
  `_rear_lower`, `bow_overhead`, `rearwindow_bow`, `SoftTop_01`–`09` (some
  with `_L` / `_Mid` / `_R`), `softtop_lowlod`, and `roof_cover_ctrl` >
  `roof_cover`. The folding animation is clip `roof` in dictionary
  `va_<model>` in `levels.rpf`, named by the model info (see
  [§11](11-vehicle-paint.md#vehicle-tables) and
  [convertible roof clips](17-other.md#convertible-roof-clips); an
  earlier search of dictionary names found nothing).
- An electric scooter has `wheelScale` 0.08 and no rim in its
  configurations, like helicopters, but it does have wheel discs (radius
  0.145). A tanker trailer body has 55 bones, none for wheels or axles, and
  no discs; which chassis carries it is unknown.

The separate [fragment physics tables](13-collision.md#134-fragment-physics)
link groups of collision shapes to skeleton bones on the two ambient rigs.

### Ped skinning: bone palette (confirmed)

Ped clothing has **8 influences per vertex**: slots 8 / 6 hold the first
four joints and weights, slots 9 / 7 a second four; the 8 weights sum to
exactly 255 on every vertex (checked on a protagonist top).

Its joint indices do **not** index the drawable's own bones. They go
through the model's **bone palette**: model +0x0C u16 count, +0x20 pointer
to that many u32 joaat bone-name hashes, sorted ascending (80 on a
protagonist top at LOD0, 28 at LOD1). Vertex joint index k is palette
entry k. Checked by body side: 110 of 110 joints land on the right side
with the palette, 50 % with the bone order.

The palette names two kinds of bones:

- the drawable's own `GM_U01_*` girth helper bones (69 on that top,
  parentless, with parent-relative binds whose parent bone is not known;
  *the same 69 on every upper garment and 51 `GM_L01_*` on every lower
  one, a shared garment rig, see
  [§12](12-materials.md#material-presets-peds)*);
- bones of a **shared ped skeleton** that is not in the clothing file: 596
  bones from `SKEL_ROOT`, with `BM_*`, `MH_*`, `RB_*` and `SEC_*` helpers,
  in the `ambient_femalebody` / `ambient_malebody` fragments of
  `packs.rpf`. `femalexxl` has the same bind pose.

Some T-shirts name 8–9 palette bones that are in neither skeleton. Heads use
another skin format (slots 8 / 9 with format 0x8c, 16 bytes per vertex):
slot 9 reads as 4 × u32 joint indices (up to 50 on a protagonist head),
slot 8 as 4 × (u16, u16) with plausible joints in the low halves, and
slots 6 + 7 weights sum to 255 on every vertex (inferred; not decoded).

**Skeleton header extras** (measured on samples): a 256-slot hash table
(u32 name hash, u32 bone index; `0xffffffff` empty) at ±0x08, and a
compressed name trie at +0x20 (prefixes such as `GM_U01_`, `L_`, `Delt_`;
encoding open). The `GM_U01_*` helper bones are parentless and their bind
pivot equals their stored translation (e.g. `C_Spine3_L` = (0.010, 0.013,
0.134)), so their bind is relative to a parent in the shared skeleton that
is not known (inferred; name-pattern guesses do not fit). The shared
skeleton's `hash16` values are not used by the palette. On vehicles model
+0x0C / +0x20 hold something else (not ascending; +0x0C = 295 on
`phoenix`). The T-shirt palette hashes in neither skeleton include
`12803912`, `1a8de2a9` and `1d2d2db7` (nine in total).

### Ped heads: face textures (confirmed)

Head drawables (`ped_standard_head`) **bind no textures**, and every head
shares one UV layout: the face in the middle, the ears at the sides and a
teeth and gums strip along the bottom. The face comes from a material
preset ([§12](12-materials.md#material-presets-peds)):

- a face preset (Category `378ad10c` = joaat("heads")) whose tag map lists the head
  (key `4bfd27dd`, joaat of the full head drawable name) is that head's
  face;
- its "base" parameter holds the **albedo in slot `ead733fc`** (1024² BC1
  sRGB) and a **BC5 normal map in slot `d689b17f`**.

Example: `cs_jada_watts_head_fr1_afr_000_y_full` (`eb9b7bee`) → albedo
`4aedb007f2b1ca3d`, normal `7742ffa7e241bd32`.

| | Heads |
|---|---|
| all head drawables | 1,657 |
| with a face through a preset (897 drawables tagged) | 893 |
| crowd heads `hc_*`: tagged, but by presets with empty texture slots | — |
| `a_head_base_*` and some `cs_*` heads (e.g. `cs_lucia`): not tagged in `packs.rpf`, and no link was found anywhere else in the install (the 188,449 meta resources of `levels.rpf` hold no presets) | — |

**Extra head maps** (2026-11-24; structure high, roles as marked): a
preset's state `Asset` (`147fdab5`) also has a keyed list `b70a22c1` of
{Key u32 hash, Item {`AssetName` = 64-bit texture hash}}, and
`TemplateName` names another preset (by `KitName`) that the preset inherits
from; chains of two occur. The head templates `c17f02fe` (male base head,
1,466 identical copies across files) and `a3bfa046` (female, 893) each list
three keys:

| Key | Texture | Reading |
|---|---|---|
| `e12585d1` | one shared texture, BC1 512 × 512, array of 3 slices; each colour channel of each slice masks one face region (9: brow, eye, cheek and mouth areas, left and right) | region masks (high); wrinkle masks (medium) |
| `85858f5e` | one shared texture, BC1 1024 × 512: red on brows, eyelids, nose bridge and chin, green on cheeks and neck, blue around the mouth | unknown (medium-low) |
| `2633887d` | one per sex; about 60 named-character presets override it with their own. None of the 61 distinct hashes is in any texture store; a search of every archive found them only inside the presets that name them | probably the wrinkle normal map (guess) |

Counts: 29,093 presets list a head drawable; of 1,300 distinct face
albedos, 1,291 resolve keyed textures (1,269 all three keys, 22 without
`85858f5e`). Face albedos are BC1 sRGB, 1024² or 256²; face normals are BC5
512 × 1024 with nothing packed in. The template's `Material` hash is in no
store either. *The 723 / 350 above count presets whose parent is each
template; the 1,466 / 893 here count copies of the templates themselves.*

The head shader `ped_standard_head` is `379efcef` and the eye shader
`ped_standard_eye` is `89a1cf72` (joaat). `ped_standard_head` has the
slots `materialctexture`, `diffusetex`, `0cd27763`, `7c0055b9`,
`0814a7fc`, `speculartex`, `diffusetexanimatedblend`, `bumptex` and
`speculartex2`; head drawables bind none of them. Open: the three key
names (not lower-case or PascalCase joaat of the obvious words), which slot
each key feeds, and what fills the specular slots at run time.

About 1,000 face albedos are ordinary textures in the `ttdb_peds_*_3`
stores. `peds_loose.ttpf` holds clothing patterns, head-space masks (beard,
brows, mouth) and red / green UV remap maps.

**Body shape** moves helper bones of the shared skeleton through a rig
([§17](17-other.md#ped-body-shape)). Other ped facts:

- `ped_standard_muv` (clothing) and `ped_standard_head` dominate; the
  `ped_standard*_blendshape*` shaders (about 300 uses) are only on rigid
  accessories (piercings, helmets, dressings).
- Hair, brows, lashes and beards are about 2 KB drawables with 3 vertices.
  The strand geometry is in separate unnamed type-3 resources
  ([§19.2](19-ped-physics-resources.md#192-strand-hair-type-3)), and the
  per-head offsets from the base heads are in
  [§19.4](19-ped-physics-resources.md#194-head-identity-deltas).

**Face preset statistics** (measured): 337 presets use a face albedo, all in
slot `ead733fc` with param "base", Category `378ad10c`; their parents are
`c17f02fe` (723) and `a3bfa046` (350), probably the two sexes. 314 face
sets have four members: the face preset, two members shared by all sets
(`9a20772d`, `adb206b7`) and one per set. 428 higher sets contain face sets;
their names look like structured ids (`125c8000`, `07dd9007`), not joaat.
Meta type `0x71` map entries are structs `00000100`–`00000106` (`6098a50e`
key + `063fa3f2` value; `00000101` = u32 key → u32 array). About 40 named
heads (`cs_lucia_head_fr1_lat_000_y_full`, `cs_kordell` …) occur, by name
or joaat, in no meta or PSO of any archive; only joaat("cs_lucia_head")
appears, in a tag-hierarchy PSO (`packs.rpf@11dd29000`, struct `00000102`:
a tag, then its parent tags). 453 of about 1,213 skin-coloured face albedos
are referenced by no preset; `story000` has no material presets.

### Animals and one-bone parts (2026-11-22)

- **One-bone palettes (confirmed).** A bone palette of length 1 (model
  +0x0C = 1) marks a **rigid part**: the vertex stream has no blend weights
  and every vertex belongs to that bone, e.g. an eagle's eyes and tongue on
  `SKEL_Head` (`34c53e77`), and props. Palette hash 0 (the arcade boss
  demon's lower body) is a collapsed placeholder.
- **Animal skeletons.** An animal's **root fragment** (`crb_eagle_bald_m_00`,
  `crb_deer_whitetailed_m_00`, `.#ft` in `packs.rpf`) holds its skeleton.
  Its parts are the drawables named `<root>_<part>_NN[_<set>]` and are
  skinned to that skeleton, not to the human ped skeleton (with the ped
  skeleton, 5,155 weights stay unresolved on the eagle).
- **Bind pose.** Animals have a 180° yaw (quaternion [0, 1, 0, 0]) on
  `SKEL_ROOT` in the bind pose that the clips do not carry, so an animated
  animal faces the opposite way from its bind pose. The same holds for
  humans (2026-11-23): clips face −Z, the bind pose +Z. The root sits above
  the feet: about 1.02 m on an adult male, 0.72 m on the white-tailed deer.
- **Shared skeleton** (2026-11-23): a character built from generic male
  parts is one skin of 814–844 joints; body locomotion clips animate 125–139
  of them (174–201 tracks), so one animated skeleton can drive many
  instances. Some idle dictionaries are empty stubs (one clip, 0 bones,
  duration about 0) next to populated `…@idle@stance_neutral` ones. The
  arcade zombie attack set
  `gameplay@melee@unarmed@arcade@zombie@standard@attack` has 8 clips of
  1.8–2.7 s on 125 bones.
- **Fur placeholders.** `*_fur_NN` parts are 3-vertex stubs, each with a
  full copy of the skeleton (white-tailed deer: 1,262 joints with fur, 296
  without).
- **LOD copies.** One part name occurs several times in `packs.rpf`, and
  the first copy is not always LOD0; the largest copy is.
- **Clips.** Animal clip dictionaries are named
  `gameplay@creatures@<land|air>@<class>@<animal>@…`. Plain `@walk`
  dictionaries hold unnamed clips (`clip_…`) with no bindings; for example
  the eagle's `…@air@bird@eagle@air@flap@medium/move` and the deer's
  `…@land@mammal@deer@normal@run@fast/move` map 91 % and 97 % of their
  tracks to joints.
- Arcade enemies in `packs.rpf`: `sc_slow_demon`, `sc_fast_demon`,
  `sc_exploder_demon`, `sc_boss_demon` and `sc_alien`, with accessories
  `schornsdemon_*`, `schelmetdemon_*`, `sc(upper|lower)accsdemon_*`,
  `scuppergarmentdemon_*` and `sc*generic`.

## 7.6 Shaders (materials)

| Off | Meaning | Confidence |
|-----|---------|-----------|
| 0x00 | u32 shader name hash = joaat of the lower-case name (`standard` = `bcf03fc1`) | confirmed (all 344 shaders) |
| 0x04 | `"atem"` | confirmed |
| 0x08 | ptr → constant block: one 8-byte pointer per constant buffer, then the buffers' data ([constants](#material-constants), below) | confirmed (2026-11-24) |
| 0x10 | ptr → per-parameter slot pointers | confirmed |
| 0x20 | ptr → parameter table: u8 ?, u8 count, …, then count × (u32 param hash, u32 data offset); param hash = joaat of the lower-case name (`diffusetex` = `c8e8c282`) | confirmed |
| 0x38..0x3F | flags, see [§12](12-materials.md) | confirmed (bytes 1, 4) |

A texture parameter's slot points to a 0x50-byte struct whose +0x30
pointer leads to a 32-byte texture reference: ptr, **u64 texture hash**
(the `ttpf` key), u32 6, u32 hash.

**Parameter table** (2026-11-22, measured on samples): header byte 1 =
texture count (17 on the four-layer shaders), byte 4 = total entries (103
there); entries are {u32 joaat name, u32 descriptor}.

| Descriptor | Meaning |
|---|---|
| `4 × index` | texture slot |
| `0x0040XXtt` | constant in the +0x08 block at float offset `XX` (`tt` = 03 or 07, probably a type) |
| `0x01YYYY07`, `0x0041XX07` | constant in another buffer (not read) |

*The two constant rows above are superseded by the descriptor layout
below: `XX` is a byte offset, not a float offset, and `tt` is the buffer,
not a type.*

### Material constants

(2026-11-24; high: a full fit on about 13,000 materials sampled in three
archives, and agreement with the shader archive's reflection tables.)

- After the texture entries, the parameter table holds the constants, 8
  bytes each: u32 name hash, u32 **descriptor** = components << 22 | byte
  offset << 8 | buffer. Bits 22 and up give the number of floats (1–4 seen;
  1 = `0x0040xxxx`, 4 = `0x0100xxxx`), bits 8–21 the byte offset inside the
  buffer, and the low byte which constant buffer (`0x03`; `0x07` on
  materials with a second buffer).
- The **constant block** (+0x08) starts with one 8-byte pointer per buffer,
  in rising order of the buffer byte, each pointing at that buffer's data
  inside the same block: with one buffer the data starts at +8, with two at
  +16. The block ends where the texture pointer array (+0x10) begins.
- Counts: 10,184 sampled materials have one buffer and 2,915 have two (the
  four-layer ground shaders, buffers of 112 and 656 bytes). Every constant
  lies inside its buffer.
- The offsets, sizes and component counts match the effect's constant
  buffer members in the reflection tables
  ([structure/06](../structure/06-models-shaders.md)), which also carry the
  type. The descriptor does not, so integer and boolean constants cannot
  be told from floats here.
- Open: why the first buffer of the two-buffer material sampled is all
  zero, and 71 sampled materials whose block lies after the texture array
  (read with a looser end bound).

Constant names recovered by dictionary search include `blendsoftness`,
`bumpiness`, `fresnel`, `puddlemaskscale`, `tintblendamount`, `tintbump`,
`tintbumpuvscalebias`, `tintintensity`, `tintroughnessbias` and
`verttintintensity`; most sit in the unread buffers. The four-layer
per-layer vec4s are in [§12](12-materials.md).

## 7.7 Vehicles

- **Wheels** are not in the car fragment. They are shared models
  `rim_<class>_NNN` and `tyre_<class>_NNN` (with `_hi`/`_mid`) authored for
  the left-front wheel at unit scale (rim radius 0.5, tyre 0.88). The game
  mirrors and scales them per vehicle. Each spawn configuration names its
  rim and tyre, and the model info has `wheelScale` / `wheelScaleRear`
  ([§11](11-vehicle-paint.md#vehicle-tables)); the 1.625 × `wheelScale`
  fit is withdrawn and what `wheelScale` measures is unknown. The tyre's
  outer radius and width are stored as [collision
  discs](13-collision.md#vehicle-fragment-bounds-and-wheel-discs). The
  wheel bones' pivots give the mount points.
- **Brake parts** (2026-11-23) are shared models in `levels.rpf` with no
  skeleton: `brakedisc_<kind>_<NNN>` (race, sport, std, track; 11 models)
  and `caliper_<N>pot_<NNN>`, `caliper_*_stock`, `caliper_rimlight` (12).
  Every brake disc has a radius of exactly 0.5 (bbox y, z −0.5 to 0.5, x
  −0.125 to 0.05; high); calipers are modelled around the same unit disc
  (extent 0.557), so both scale by the wheel's [brake-disc
  radius](13-collision.md#vehicle-fragment-bounds-and-wheel-discs) ÷ 0.5
  (medium that the caliper shares the disc's space unrotated). A
  configuration names at most one of each: of 4,063, 2,030 name both, 228
  only a disc, 14 only a caliper and 1,791 neither.
- **Hinged panels** (names high; hinge line medium, two cars): bones
  `door_dside_f`, `door_dside_r`, `door_pside_f`, `door_pside_r`, `bonnet`
  and `boot` sit on the hinge line, and the panel's vertices lie to one side,
  which gives the opening direction (one hatchback's bonnet hinges at the
  windscreen end, one convertible's at the nose). Doors have a
  `door_droop_<side>` parent. Open: opening limits, and hinge axes that are
  neither vertical nor lateral.
- **Lamps** (2026-11-23, high: bone and shader names plus skinning on three
  vehicles): a lamp is up to three meshes on its own bones: a reflector
  (`standard_vehicle_2lyr_reflector`, `03b27476`) skinned to the lamp
  bone, a lens (`standard_vehicle_alpha`, `389206d1`) skinned to
  `<lamp>_glass`, and on some lamps an LED mesh
  (`standard_vehicle_emissive_animated_bones`, `29648098`) skinned to the
  lamp bones. Lamp bones seen: `headlight_l` / `_r`, `headlight_hi_l` /
  `_r`, `headlightcase_lf` / `_rf`, `drlight_lf` / `_rf`, `taillight_m`
  (and `_l` / `_r`), `brakelight_l` / `_r`, `reversinglight_l` / `_r`,
  `indicator_lf` / `_lm` / `_lr` / `_rf` / `_rm` / `_rr`, `platelight`,
  `bootlight_ls`, `doorlight_lf` / `_rf`, `interiorlight`, each with a
  `_glass` partner where it has a lens. On one sports car one 485-triangle
  LED mesh is skinned across 13 lamp bones; on a saloon the LED mesh covers
  only the two `drlight` bones. How the game switches a lamp is not decoded
  (the shader name suggests the lamp bones are animated; no clip or value
  was read).
- **Light patterns** (2026-11-23; structure medium, meaning low): struct
  `0f93517e` (157 in the vehicle library in `data.rpf`): name, slot
  `37a6f9ef`, a half float `699dd4d9` (1.0 in the sample; the struct also
  has a field whose hash is `timeMultiplier`), and a list `d3d4919b` of
  `69a4bdea` {`395039a9` joaat of a lamp bone name, u64 flags, list
  `a478703e` of 6-byte steps `67cbceee` {`a13c79ef` half float, `fc46055f`
  u16 (0 or 1), `96e5b9a7` u8}}. 571 distinct lamp names occur; 200
  resolve to vehicle bone names (`auxlight_*`, `bootlight_*`,
  `brakelight_*`, `drlight_*`, `foglight_*`, `headlight_*`, `indicator_*`,
  `doorlight_*`, `int_ceiling*`, `dashglow` …). Lamps have 1–29 steps.
  Probably on / off sequences (sirens, indicators, strobes); not played
  back, so the timing unit and the flags are unknown. 54 vehicle models have
  siren or strobe-light bones.
  *Refined (2026-11-23; structure high: 339 of 373 vehicle records resolve
  patterns through the chain below, and ambulance lamps light on the right
  bones):*
  - Pattern `0f93517e` (160 bytes): name, slot, clock `1b953013` (u64; only
    two values seen, one on siren patterns, one on ordinary lamps), speed
    `699dd4d9` (f16, mostly 1.0), lamp list `d3d4919b`. Where: 157 patterns
    in the vehicle library, 178 in a shared pattern pack, and small light
    packs per emergency vehicle (1–10 patterns each).
  - Step `67cbceee`: `a13c79ef` f16 **phase offset** (mostly 0–1, a few
    above 1), then the **flash type** number: in the packs a u8 `5b92fa4b`
    (4-byte struct), in the library a u16 `fc46055f` plus u8 `96e5b9a7`
    (6-byte struct). 29 type numbers are in use (0–38); 1 is by far the most
    common, indicators use 6.
  - Link chain: configuration part `0caef606` → list `61e1e572` (`features`) →
    `da592d57` {`1ff6a884` pattern names, `dd466dfe` another name list,
    probably lamp sets} → pattern.
  - Lamp set `865400aa` {list `804fdca7` of `ba860a07`}: `079baba7` (`attachBone`) u64 whose
    low 32 bits are the bone name hash, `066532dd` light archetype name hash,
    plus an archetype u64 (name hash in the high half, a second hash in the
    low half). Archetype names found by guessing: `red`, `amber`,
    `halogen`, `halogen_warm`, `led`, `led_warm_white`, `warm_white`,
    `indicator_amber`, `amber_roof_spotlight`; 117 more are hashes only.
  - **Lamp colours** (2026-11-23; links high: 19,799 of 19,808 vehicle
    lamps resolve to a light set by exact id and name matches). *Correction
    to the bullet above:* `066532dd` is the **light set** name hash (the
    names listed above are set names), and `ea42cb63` is the light
    archetype reference. In the vehicle library it is type 0x4d: high
    32 bits the set name again, low 32 bits the archetype's name hash. In
    the packs it is type 0x4f, the archetype's 64-bit id.
    - The light archetypes are ordinary archetypes (`82d6fc83`) in the
      levels archive: 64-bit id `8bfb6095` at +0 (repeated in `99d56b7b`),
      name hash at +0x74. Their extension is `7af058f7` (not the usual
      light extension `27922c43`) and holds named light sets `76402c3c`
      {name hash, list `f54b227b` of light attribute records}.
    - Each light: colour u8[3] `5bdc13e3` (two more u8[3] fields,
      `8822fbce` (`volOuterColour`) and `3a615dea`, are `ffffff` here), intensity float
      `efcd993d`, and a flash block `e9476573` {type u8 `5b92fa4b`, speed
      `699dd4d9`, offset}, the same fields as the pattern steps above.
    - All 55 light archetypes the vehicles use are in one resource (805
      named sets, 2,319 lights). One set name can have different colours in
      different archetypes (70 of 720 names), so both halves of the
      reference matter. Example: an ambulance's three siren sets are white,
      intensities 1000, 2600 and 4000, so the colour seen in game must come
      from the lens.
  - Open: what each flash type looks like over time (no table found in the
    meta resources; *a later search, 2026-11-24, also found none, counting
    26 type numbers in 217 patterns, 168 on one clock and 49 on the other;
    the count differs from the 29 above, probably from a different set of
    patterns*), the cycle length, the two clock values; whether the
    light colour bytes are linear or sRGB, and where the lens colour comes
    from.
- **Steering** (medium): `wheel_turn_lf` / `wheel_turn_rf` are the parents
  of the front wheel bones (on a motorcycle `wheel_turn_lf` is the fork); the
  steering wheel is bone `steeringwheel` under `steeringcolumn`. Model info
  `steerWheelMult` is 0.7 on a sports car. The steering lock is the handling
  record's `fSteeringLock` (in a PSO, [§17](17-other.md#vehicle-handling)).
- **Damage parts**: separate fragments `<car>_dmg_mod_<part>_NNN`.
  **Mod-kit parts**: `<car>_<part>_<kit>`. Both with `_hi`/`_mid`.

## 7.8 Weapons (modular)

(2026-11-23.) **Location** (high, counted from the index): `data.rpf` holds
2,855 models named `w_*` for 95 base weapons (by class word: ammo 16, me 14,
at 8, smg 7, pi 6, sg 6, ar 5, lr 5, rv 5, sports 5, ex 4, kit 4, sr 4, tr 4,
mg 2). `levels.rpf` has only 19 placeholder shapes (`w_preset_*`,
`wpn_proxy_*`, `w_root_01`).

**Structure** (high): a weapon's own model, `w_<class>_<make>_<NN>` (plus
`_hi` / `_mid`), is a skeleton with a shell of about 60–150 triangles (an
assault rifle's: 4 bones, 148 triangles, 14 KB). The visible weapon is
separate part models `<weapon>_<slot>_<NN>` with the same LOD suffix; on that
rifle: receiver, barrel (3 variants), handguard (3), stock (3), mag (3),
dustcover (2), sight_r (2), bolt, pgrip, trigger, muzzle and follower. Part
`_hi` models run up to 1.4 MB (the receiver: 9,173 triangles).

**Attach points** (rule high: eleven weapons assemble with it, a rifle,
pistols, shotgun, SMG, revolver, sniper rifle, launcher, machine gun,
machete and grenade; bone lists from one rifle and one pistol):

- A part's root bone is `AAP_<Slot>`, at its origin; the model it mounts on
  has a bone `WAP_<Slot>`. Names match case-insensitively (`WAP_Dustcover` /
  `AAP_DustCover`), and the AAP name can differ from the file's slot word (a
  rifle's pgrip part has `AAP_Grip`).
- A part with no skeleton is modelled around its attach point and takes the
  `WAP` bone named after its slot.
- Rifle: the base has `WAP_Receiver`; the receiver has `WAP_Barrel`,
  `_Bolt`, `_Dustcover`, `_Grip`, `_Handguard`, `_Mag`, `_Sight_R`, `_Stock`
  and `_Trigger`; a barrel has `WAP_Muzzle`; one handguard has
  `WAP_Rail_Front_Bot` / `_Side_1` / `_Top` and `WAP_FrontGrip`; a dustcover
  variant has `WAP_IronSight_R` and `WAP_Rail_Optics`.
- Pistol: the base has `WAP_Frame`; the frame has `WAP_Barrel`, `_Mag`,
  `_Magwell`, `_PGrip`, `_Slide` and `_Trigger`; the slide has
  `WAP_Sight_F`, `_Sight_R` and `_Muzzle_Slide`.
- Other bones: `Wpn_Root`, `Wpn_Main`, `Wpn_Holster`, and `Wpn_LOC_*` /
  `Wpn_Anim_*` pairs (bolt, cock, sling loop).

**Shader `weapon`** (`e68a4a98`): binds `speculartex`, `bumptex` and
`speculartex2` and no colour map (66 of 72 part materials looked at). Both
specular maps are linear BC7 data (1024² on a stock); the second's red
channel looks like a metal mask. The colour probably comes from a material
palette (medium, inferred from the missing colour map). *Later look at one
pistol part (2026-11-23; contents high, roles low):* every channel is
greyscale data. `speculartex`: R dark surface detail, G detail centred on
0.5, B flat 0.5, A an occlusion-like mask. `speculartex2`: R sparse wear
specks (not read as a metal mask here; not reconciled), G broader dirt /
wear, B centred on 0.5 (height-like), A 255. The 0.5-centred channel is the
same pattern as the vehicle layer detail maps, which suggests a 0.5-centred
channel picks a position along a colour ramp (inferred; the shaders are
native code). The colour data itself was not found.

**How the weapon shader colours** (2026-11-23; read from the deferred pixel
shader's machine code; mechanism high, metal medium, block field meanings
partly unknown). This confirms the gradient-map reading above and replaces
the material palette guess:

- Textures: `speculartex`, `20245319`, `bumptex`, `speculartex2` and an
  optional `materialid` (BC4). There is no colour texture.
- Sub-material index = min(7, floor(63 × `materialid`.r)), or a flat
  per-vertex uint when there is no `materialid` texture (*2026-11-23,
  medium, from data:* weapon parts bind no `materialid` texture; the index
  is in the green byte of the vertex colour stream (slot 1) as 4k + 2. 60
  of 62 parts of three weapons hold only such values, and the indices
  used match the distinct colours of each part's preset), so a weapon
  material has up to 8 sub-materials.
- Everything else comes from one parameter block per material in a raw GPU
  buffer, read at base + index × 16: arrays of 8 entries of 16 bytes,
  128 bytes apart, from +16 to about +3472. Seen: +16 flags, +144 a scale,
  +1936 a texture index per sub-material, +2064 / +2192 / +2320 three RGBA
  byte colours, and more float3 sets for two overlay layers.
- Colour is a three-stop gradient over a grey value g (the green channel
  of the sub-material's texture):

  ```text
  g < 0.5:  grad = lerp(low, mid, 2 g)
  else:     grad = mid + (high - mid) * (g - 0.5)
  rgb = grad * a + (1 - a) * g       # a = alpha of the mid colour
  ```

  So each sub-material has three tint colours applied as a gradient map,
  and the textures are greyscale data.
- G-buffer: as for vehicles, diffuse = base × (1 − metal), with no
  metalness channel.
- Open: which game data fills the parameter block, and which of the two
  upper colour fields is mid and which is high.
- *Negative result (medium):* the two material name hashes used by weapon
  component lists (`b5e90fae`, `228565d0`) occur in no decoded PSO or
  script document of the script archive, the story script pack or the
  common archive, and a 1–3 word name search found no name. *Extended:*
  they occur in no meta resource and no PSO file of any indexed archive.
- **Weapon colours come from layered presets** (2026-11-23, high): the
  per-component asset struct `85492dc3` field `72f5e33e` and the variant
  list `6724161c`.`14b064c2` hold keys of the weapon
  [layered material library](12-materials.md#layered-material-library).
  `85492dc3`.`873ef899` is joaat of the component's model name (972 of
  1,006 resolve), and `6724161c`.`c763edb4` names the part a variant
  recolours (frame, body …).
  The Components list (struct `b8784be8`: id, material, byte `593b09b0`)
  is not the colour source: its two material values (the hashes above)
  match nothing in either library, and their role is still unexplained. Where weapon
  tint colours are defined is unknown.
- **Finishes per part type** (2026-11-23, high, from data): in a weapon
  family file each `6724161c` variant is a finish for one part type,
  `c763edb4` = joaat of the part word and `14b064c2` = its layered preset;
  48 families, 407 part-type lists, 1,300 variants. 36 of 54 part words are
  named (each re-hashed), among them `barrel` `13f890a9`, `receiver`
  `aa146920`, `stock` `3c0dd756`, `magazine` `da748070`, `frame`
  `963ad5a5`, `trigger` `c3afd061`, `bolt` `d278cb8a`, `pgrip` `0d8d8163`,
  `muzzle` `3af92b85`, `grip` `6fad1258`, `slide` `74e3d185`, `optic`
  `1301cc6f`, `handguard` `60a91853`, `dustcover` `a9bccac1`, `body`
  `8c255aa8` and `pump` `66eafecf`. Open: which finish a weapon has by
  default, the variant names, and 18 part words. *Later (2026-11-24): 53 of
  54 part words named; new: `optics` `879e7ea8`, `sight_optic` `6a758e81`,
  `optic_rail` `8a34e978`, `carry_handle` `8e73a6a7`, `mag_assist`
  `c7b6710c`, `saddle` `1d2fedf7`, `accs` `3a998c68` (each re-hashed).
  Unnamed: `060dc09e`, on fore grips. The default finish is answered
  below.*
- **How finishes link to weapons** (2026-11-24; chain and default high,
  from full-population counts; condition reading medium, from structure
  only). Weapon family PSO files hold:
  - Weapon `bc92474a` (168 records): `20fa4150` is the part set name (0 =
    use the weapon type's), `WeaponType` names a record `454c2e3b` (112)
    whose `20fa4150` is the part set, `43ab621f` is a list of mod names
    and `b5fbecab` the default mod name (non-zero on 72 of 168).
  - Part set `5ad6772f` (113): `Entries` of `ffddc90c` {`5cceffc2` part
    type hash (the part words above), `Options` of `19962073`
    {`cac8083d` component record name (`85492dc3`), `Condition`}}.
  - Condition `38d9ad3a` {`285cceca` mod name}: the option applies when
    that mod is fitted (1,247 options). `e6b9cf20` and `fed5f24b` hold
    `Entries` of conditions, probably and / or (which is which is not
    checked).
  - Mod `a5791c62` (3,338): `72f3252e` category hash, `c897af02` (`Features`) list of
    record names. Category `0ee68bd5` is the finish mods, listing variant
    records `6724161c`; category `1f185279` lists `349a32ad` records (a
    `Mods` list plus `ba466b50` weapon) or nothing.
  - Variant `6724161c` (1,301 here, 1,300 in the count above):
    `c763edb4` part type hash, `14b064c2` layered preset key.
  - Weapon preset `8d690ccf` (825): `Mods` list and `ba466b50` weapon
    name; 111 include a finish mod.

  Checks: all 1,299 (weapon, variant) pairs find the variant's part type
  among that weapon's part set entries, and 1,276 of 1,301 variants reach a
  model this way (621 models). **Default finish:** the default mod is of
  category `1f185279` in all 72 cases and lists nothing in 71, so a weapon's
  default look is each component's own preset (`85492dc3`.`72f5e33e`).
  Open: the variant names, the other mod categories, the entries' `Flags`
  values, and which condition struct is and / or.

Open (*the default variant of each slot and the source of part colours are
answered above: each component's own layered preset*): what mounts on the `WAP_Rail_*` points and how the shared
`w_at_*` attachments choose them; where a magazine follower and a pistol's
reflex mount attach (no matching `WAP` bone on the builds tried); how a
weapon model is tied to its display text (the text tables hold weapon names,
e.g. one under label `2bd414b5` matching a pistol's model word, but no 0x7a
resource or decoded PSO in `data.rpf` holds a weapon model's name hash or
that label, and the label is not joaat of the model name or obvious
prefixed forms).

## Open questions

- Normal top 12 bits; slots 12, 13, 17.
- The parent bone of the `GM_U01_*` helpers; the head skin format (0x8c);
  palette bones found in neither skeleton.
- The face of heads no preset lists (`a_head_base_*`, some `cs_*`), and of
  crowd heads whose presets have empty slots.
- Occluder and water-vegetation LOD buffers (another format).
- The other constant buffers (`0x01YYYY07`, `0x0041XX07`), the meaning of
  `tt`, most constant names (cable thickness and paint constants are
  probably there; unchecked).
- The 64-bit texture name hash; 152 of 229 parameter names.
- How damage parts are swapped in; how `wheelScale` sizes the shared wheels.
- The normal encoding of some meshes (a head and a helmet) fits no axis
  order (best score 0.5–0.7, though 99 % of their normal words look valid).
- Why some vehicle skeletons stop early: `dominator_hi` (72 bones) and
  `trotter_hi` / `_mid` (107) end after `cargo_volume`, `chassis_dummy` and
  `chassis_lowlod`, though their strings hold the door and wheel bone names
  (typical cars have 270–390 bones; 3 of 365 bodies). *Probably the
  bone-name character set above (a reader stopping at a dashed name);
  inferred, these three not re-checked.*


---

← [6. BCPack](06-bcpack.md) · [Index](README.md) · [8. Meta resources (type 0x7a)](08-meta.md) →
