# 19. Strand hair, cloth and head deltas (unnamed `packs.rpf` resources)

*Part of the [SixNG format specification](README.md). Numbers as of 2026-11-22; confidence: confirmed / inferred / unknown.*

`packs.rpf` holds 47,847 RAGE resources that carry no `name.#xx` string
and that no other page of this specification covers: not drawables,
fragments, bounds, navmesh tiles or clip dictionaries. Together they
inflate to 3.13 GB, about 32 % of the archive. This page splits them
into families and decodes three of them in full: strand hair, verlet
cloth and per-head identity deltas.

## 19.1 Family split

All of them are ordinary resources ([§2](02-resources.md)): a system page,
the page map pointer at +0x08 and in some cases one graphics buffer. The
byte at **+0x1A** is a type, and the u32 at **+0x1C** is either 0 or the
constant **`7d8f3db3`**. Types 1, 2, 3, 4 and 5 all use that constant.
Its name is not known.

| Family | Key | Count | Inflated | Share | State |
|---|---|---|---|---|---|
| strand hair / fur | type 3, `7d8f3db3`, one GPU buffer (flags 0x1380) | 20,010 | 1,083 MB | 34.6 % | **decoded** ([19.2](#192-strand-hair-type-3)) |
| verlet cloth | type 2, +0x1C = 0, no GPU buffer | 12,364 (12,365 in a second count) | 401 MB | 12.8 % | particles and edges **decoded** ([19.3](#193-cloth-type-2)) |
| soft body | type 2, `7d8f3db3` | 702 | 39 MB | 1.2 % | same layout as cloth |
| head identity deltas | 240-byte system page, one 0x1601 buffer, base-head hash at +0x58 | 6,758 | 316 MB | 10.1 % | **decoded** ([19.4](#194-head-identity-deltas)) |
| type 5 (collision primitives?) | type 5 | 6,388 | 6 MB | 0.2 % | partly ([19.5](#195-other-families)) |
| type 1 | type 1 | 480 | 1 MB | 0.04 % | names only |
| `F` payloads | type 0, 176-byte system page, one 0x1380 buffer, u64 key at +0x20 | 490 | 1,273 MB | 40.7 % | layout only ([19.5](#195-other-families)) |
| `E` headers | {u32 n, u32 n, ptr} at +0x10, u64 key at +0x48 | 546 | 8 MB | 0.3 % | layout only |
| other | | 109 | 1 MB | | |

Resources of one family sit together in the archive. The head delta
runs follow their head drawables (one run after each `head_<class>_*`
group). Hair, cloth and the other families come after the last named
resource.

## 19.2 Strand hair (type 3)

Each resource is one **guide-strand groom**: hair, beards, brows, lashes,
stubble and animal fur. Strand positions are model space, Z-up, metres,
and line up with the ped head drawables without any transform. A curly
fringe and a full beard sit on `head_mr1_base_000_m_full` (checked by
eye). The bones the grooms name confirm what they cover. Across the
family the most common are `BM_Head` (24,074 references; 11,582 of them in
the +0xA0 arrays, so the total probably includes the +0xB8 hashes),
`BM_Occipitalis`, `BM_Neck1/2`, `SKEL_HEAD`, `EarRig_*`, `FACIAL_C_Jaw`,
`BM_Chin`, `BM_L/R_Cheek`, and for animals `SKEL_Tail0..5`,
`SKEL_*_THIGH`, `SKEL_*_CALF`.

The spec has said that hair is "built at runtime" from 3-vertex
placeholder drawables ([§7.5](07-models.md#ped-heads-face-textures-confirmed)).
The geometry is in these resources. No direct link from a placeholder
drawable to its groom has been found (no shared hash, see open
questions).

### System page

| Off | Type | Meaning | Confidence |
|---|---|---|---|
| 0x00 | u64 | 0 | confirmed |
| 0x08 | ptr | page map | confirmed |
| 0x10 | ptr | 0x10-byte block, or 0 (non-zero on 526 resources) | layout confirmed, meaning unknown |
| 0x19 | u8 | flags (0x00, 0x80, 0x81 …) | unknown |
| 0x1A | u8 | type = 3 | confirmed |
| 0x1C | u32 | `7d8f3db3` | confirmed |
| 0x20 | u32 | flags; the same value is at gfx +0x74 (1,500/1,500) | confirmed (copy) |
| 0x30, 0x40 | vec4 ×2 | **root bounding box** min / max. w = ∓ root radius | confirmed: 100 % of strand roots inside |
| 0x50, 0x60 | vec4 ×2 | second box (FLT_MAX / −FLT_MAX when empty). 92 % of points are inside it | inferred |
| 0x70 | ptr | buffer descriptor (0x40 bytes, as [§7.1](07-models.md#71-buffer-descriptor-system-page): +0x08 size in dwords, +0x10 flags 0x1380, +0x18 graphics pointer, +0x28 pointer to a 0x20-byte block) | confirmed |
| 0x78 | u64 ×3 | 64-bit hashes; zero on 99.7 % | unknown |
| 0x98 | u32 n, u32 cap | | confirmed |
| 0xA0 | ptr | **u32[n] bone joaat hashes** (the bones the groom follows) | confirmed (names resolve) |
| 0xA8 | u32 n, u32 cap | | confirmed |
| 0xB0 | ptr | n × 12-byte part records {u16, u16, u32, u32}. The first record repeats +0xD0..+0xDB | layout confirmed |
| 0xB8 | u32 ×4 | bone joaat hashes | inferred |
| 0xD0 | u32 ×2 | counts | unknown |
| 0xDC | u16 T, u16 S | root-triangle count, guide-strand count (equal to gfx +0x5A / +0x58 on 1,500/1,500) | confirmed |
| 0xE0–0x13F | f16 / u8 | groom parameters. 0x110 is a u8[16] profile, often 0xFF→0x00 (a taper) | inferred |

The header, descriptor, 0x20 aux block, bone array, part records and
page map **tile the whole system page** on all 20,010 resources.

### Graphics buffer

A 0xC0-byte header followed by typed sections. S = guide strands,
K = points per strand, T = root triangles.

| Off | Type | Meaning | Confidence |
|---|---|---|---|
| 0x00 | u32[8] | section offsets A0–A7 (0 = absent) | confirmed |
| 0x20 | u32 ×3, u32 | 0, 0, 0, 0xFFFFFFFF. *A later sample reads section offsets C0–C2 here, not zeros (measured on samples; not reconciled)* | confirmed (first reading) |
| 0x30 | u32[8] | section offsets B0–B7 | confirmed |
| 0x50, 0x54 | u32 ×2 | 0x4E8 on most resources; ?. A u24 at +0x55 = S × K (measured on samples) | unknown |
| 0x58 | u16 | **S** | confirmed |
| 0x5A | u16 | **T** | confirmed |
| 0x5C | u32 | **K** (2–50; 3 on 5,001 resources, 30 on 1,690) | confirmed |
| 0x60 | f16 ×6, u8 ×4 | parameters | unknown |
| 0x74 | u32 | copy of system +0x20 | confirmed |
| 0x80 | u32 ×4 | small counts (0–7) | unknown |
| 0x90 | u8[16] | monotonic lookup table | inferred |
| 0xA8 | u32 | 3 | inferred: records per tail group |
| 0xAC | u32 | offset of the tail records, or 0 | confirmed |
| 0xB0 | u32[4] | section offsets Q0–Q3 | confirmed |

Sections. Each starts on a 4-byte boundary. Sizes are exact up to that
padding.

| Section | Size | Contents | Confidence |
|---|---|---|---|
| **B5** | S × K × 8 | **points: 4 × f16 (x, y, z, w)**, strand-major. w is non-zero at the root on 99.6 % of strands; first read as a **width** (about 0.57 mm along the strand, about 3 mm at the root); a later sample finds **w = 0.1 × rest length** (B2), so it is probably not a width | xyz confirmed; w inferred |
| **B4** | T × 6 | **root triangles: 3 × u16 strand indices** (all < S) | confirmed |
| **B0** | S × 2 | **f16 strand length** (equals the polyline length to 2 % on 98 % of resources) | confirmed |
| A2 | S × 4 | 2 × f16: probably the **root UV**. Present on 250 of 250 sampled grooms, 73–100 % in 0..1, and it follows the root position (median \|r\| 0.86 for u, 0.83 for v) | inferred |
| A0 | S × 4 | per-strand u32; byte 0 = clump size (see B6) | inferred |
| A1 | S × 4 | per-strand u32: root triangle index | inferred |
| A3 | S × 4 | constant fill | measured |
| A4, A5 | S × 8 each | **root skinning: 8 bone indices and 8 weights summing to 255** (448 of 448) | measured on samples |
| A7 | S × 8 | per-strand 8 bytes | unknown |
| A6 | S × 12 | per-strand 12 bytes | unknown |
| B1 | S × K × 4 | per point, 2 × f16: simulation curves | inferred |
| B2 | S × K × 2 | per point, f16 **rest lengths**: point 0 = strand length, point k = segment length (not the arc length) | measured on samples |
| B3 | S × (K − 1) × 4 | per segment: a unit vector perpendicular to the segment | inferred |
| B6 | T × 4 | per triangle, 4 × u8: **c0 = strands grown in that triangle** (Σ c0 = sys +0xD0 on 121 of 134 densest LODs; tracks triangle area, r 0.97–1.0); c2 ≈ clump count and c1 ≈ c2 × clump size (Σ c1 within 1 % of sys +0xD4 on 90 of 134, inferred); whiskers c0 = 0, c1 = c2 = 1. Totals are equal across a piece's LODs; low LODs saturate at 255 | c0 measured on samples; c1, c2 inferred |
| B7 | T × 1 | per triangle, u8 | unknown |
| Q3 | S × 4 | root normal, n = 1 − b / 127.5 | measured on samples |
| Q0, Q2 | S × 4 | per strand | unknown |
| Q1 | 4 | u32 | unknown |
| (implicit, after the last table section, on 26 % of resources) | 2 × S × 4 + 8 + 4n | two per-strand u32 arrays, then a 2 × f16 range and n × {f16 value, u16 monotonic 0..0xFFFF} | inferred |
| tail (offset at 0xAC) | 4 × (multiple of 3) | records {u24 index, u8 weight}. The record count is always a multiple of 3, and the three weights sum to 255 on 62 % of resources: extra points blended from guide points (143 of 205 fit) | inferred |

**Rendering and parameters** (2026-11-23; measured on 484 sampled grooms,
not re-checked):

- System +0xEA and +0xEE both hold the longest strand; +0xF0 is probably the
  strand width or radius (0.2–0.5 mm, zero on 29 %) with its taper at +0x110
  (inferred). Simulation settings sit at gfx +0x60–0x9F and system
  +0xE8–0x132 (located by position only).
- The drawn count is probably c0 + c1 per root triangle (a cat body:
  277,789 + 86,802 strands from 6,757 guides; inferred). Growing that many
  random strands per triangle raised the covered area on three cat pieces
  from 4.2 / 1.7 / 6.4 % to 88.2 / 90.7 / 65.4 %. The same covers the
  coats of a cat, chimpanzee, boar, bear, boxer and deer, which supports
  reading B6 as strand counts (a visual check only, not a comparison with
  the game). Hair on people is unverified: a hand-picked scalp groom did not
  line up with its head, and there is no automatic hair link for people.
- The hair effects are an `smh_*` family (pose, simulate, `smh_gen_N`
  "InterpolateStrands" in 80 variants, modify, draw, a compute rasteriser),
  then `hair_resolve` and the `TL_PS_PBR_MATERIAL_HAIR` lighting passes;
  the shader code is native, so no constant or texture names can be read.
- **Fur colour:** all 601 fur presets bind the same 16 × 16 marker texture
  (`1ae2616f99a5e7ae`) and a 4 × 16 black normal, so the colour probably
  comes from the body skin at the root UV (inferred). No drawable uses
  `ped_standard_hair` or `ped_standard_fur`. `common.rpf` has only
  time-cycle hair values.

Decoding the points:

```
P = f16[S][K][4] at gfx + B5
strand s, point k: (x, y, z) = P[s][k][0..2]   # Z-up metres, model space
```

**Checks over all 20,010 resources**:

- the typed sections tile 100.0 % of the 902,231,288 buffer bytes, with
  no gaps or overlaps. Another 170 MB of page padding after the buffer
  is not counted;
- the system pages are 100.0 % tiled (11,121,744 bytes);
- 99.9 % of strands are continuous: no segment is longer than 4 × the
  strand's median segment;
- 100 % of roots are inside the root box.

Totals: 5.57 M guide strands and 35.3 M points.

**Independent check** (a second decoder written from this page, all of
`packs.rpf`, 2026-11-22): 20,010 grooms decode with 0 errors, 5,565,061
strands and 35,297,629 points; every root lies inside the sys +0x30 / +0x40
box; strand continuity has a median of 100 % per groom (40 grooms are under
90 %). Front and side views show hair, sideburns and animal fur where
expected. Bone hashes are lower-case joaat: the named fragment skeletons
and head drawables name 97.4 % of the 391,179 bone references.

**Grooms in other archives** (2026-11-23, confirmed by scan):
`pack/core_restofworld` holds 36 grooms in 13 pieces, all rooted at the
pelvis (z about −0.1). There are none in `pack/story000` (20 GB, 41,799
unnamed objects scanned), `story000_restofworld`, `core_restofworld001`,
`shared_minigame000` or `data.rpf`; `levels.rpf` was not scanned to the
end. `player_story000_00`'s assets list 58 distinct hair piece ids, but its
block of `packs.rpf` (between the leg-hair pieces and
`player_story000_01`'s) holds about 24 pieces: brows, a bob (3 assets), a
buzzcut, two pixie cuts and lashes. Its four ponytail assets (6, 8, 8 and 7
pieces) have no files in any scanned archive (confirmed by count; inferred
that the files are not in this install).

**LODs (inferred):** neighbouring resources often hold the same groom
with fewer strands. Where several resources in a row share the stored
second box, S falls strictly from one to the next in 1,201 of 1,266
cases, by a median factor of 0.60. Example: S = 141 → 85 → 52 → 31 at
K = 26. Grouping consecutive resources by an equal second box gives 18,551
groups for the 20,010 grooms, so only a minority sit in such runs. A better
rule is consecutive grooms with the **same bone set and strictly fewer
strands** each time: 5,915 chains.

**Regions (inferred, from bones and the root box):** head-only grooms whose
roots all lie below z = 0.74 and in front (y < 0) are beards and moustaches,
not scalp hair. By chain head: scalp 1,246, face 2,655, brows and lashes
627, animal 1,168, body 177, other 42. In archive order, runs of face
grooms are each followed by one scalp groom, perhaps one set per character,
but nothing names them. Example: the scalp groom at `packs.rpf@10950d010`
and the beard at `packs.rpf@f14aa010` sit on `head_mr1_asi_052_o_full`.

*Correction (2026-11-22):* the "animal" region (bones `Tail`, `Paw`,
`Wing`) misses animal head and body pieces without those bones, and the
first "animal" chains have finger bones and a 1.2 m span, so they are
probably not animals. The "same bones, fewer strands" rule also splits some
pieces in two. The counts above are kept as measured.

**No link to peds.** The 681 hair, beard, brow and lash placeholder
drawables (1,712-byte system page) hold only joaat of their own name at
sys +0x6A0 plus constant hashes, and none of those hashes occur in any of
the 47,847 unnamed system pages. There are about 6,000 chains for 681
placeholders.

## 19.3 Cloth (type 2)

Verlet cloth: flags, canopies, awnings, tarps (1–6 m across) and garment
pieces. The soft-body variant (+0x1C = `7d8f3db3`, 702 resources) uses
the same layout. It names body helper bones (`BM_pelvis`, `BM_Stomach`,
`SEC_*_Full_Breast`, `GM_U01_C_Spine4_LF`, `MH_*_HipFront` …); this is
the soft-body data that [§17](17-other.md#ped-body-shape) mentions.
Hashes inside the cloth resources include `default` and `Legacy`
(tuning profiles) and garment names such as
`Gmt_Jacket_Blazer_F_Hang_01b`.

| Off | Type | Meaning | Confidence |
|---|---|---|---|
| 0x08 | ptr | page map | confirmed |
| 0x1A | u8 | type = 2 | confirmed |
| 0x1B | u8 | varies (0, 127, 231, 50 …) | unknown |
| 0x20 | u32, u32 | `0x8000000x` flags, 0x16 | unknown |
| 0x28 | ptr, u16 n, u16 cap | **particles: n × (f32 x, y, z, w)**. w is per-particle (e.g. 2.9–13.4), probably mass; a second count found values from the hundreds to 1e5, so the meaning is open | xyz confirmed; w unknown |
| 0x38 | ptr, u16 m, u16 cap | **edges: m × {u16 a, u16 b, f32 rest length, f32, u32 flags}**. The second f32 is often constant per resource (stiffness?); flags are 0 or values such as `0xFFFE0000` | confirmed (layout) |
| … | {u32 n, u32 cap, ptr} arrays | per-particle 22- and 40-byte records, u16 and u32 index lists, a 400-byte block. A second particle / edge pair follows at +0x2C8 / +0x2D8, on all resources: a smaller set, partly the same points as the first, whose rest lengths match less often (39–98 %) | layout inferred; second pair confirmed |

**Checks over all 13,066 resources:**

- all decode, and edge indices are < n on 100 %;
- the edge rest length equals the distance between its two particles in
  the stored pose (within 1 mm + 0.1 %): median 85 % of a resource's
  edges for cloth and 97 % for soft bodies. ≥ 50 % of edges match on
  91.9 % of cloth and 96.4 % of soft-body resources;
- totals: 1.00 M particles and 4.45 M edges.

**Independent check** (2026-11-22): 13,067 resources decode with 0 errors
(12,365 cloth and 702 soft bodies, one more cloth resource than above).
The first set holds 1,002,787 particles and 4,454,664 edges, and the rest
lengths match on a median 84.7 % (cloth) and 96.9 % (soft body) of edges,
as above. The second set holds 836,718 particles and 2,745,856 edges.
Hashes in the pages resolve to `default` (all), `Legacy` (12,258) and, on
soft bodies, ped body bones; 390 name a garment (`Gmt_*`).

A soft-body cage drawn from its edges shows a ring-and-cross-brace
lattice around the hips (stretch, shear and bend constraints).

Typed so far: particles + edges are 20 % of cloth system bytes and 17 %
of soft-body bytes. The rest is mapped only as arrays of known element
size.

## 19.4 Head identity deltas

One resource per head drawable **per LOD**. Each stores, for every
vertex of the class's base head, the offset from the base head to that
head:

```
head_vertex = base_vertex + delta
```

Example: `head_fr1_afr_000_m_full` minus `head_fr1_base_000_m_full`.

| Off | Type | Meaning | Confidence |
|---|---|---|---|
| sys 0x08 | ptr | page map (0x500000D0) | confirmed |
| sys 0x10 | ptr | 0x50000070 → buffer descriptor at 0x70 (flags 0x1601, as a mesh blob) | confirmed |
| sys 0x58 | u32 | **joaat of the base head drawable**: `head_fr1_base_000_m_full` (`9d2a8e38`), `head_mr1_base_000_m_full` (`7632be11`), `…_lod2`, `…_lod3`; `fo1`, `fb1`, `mo1`, `mb1`, `mi1`, `mk1` likewise. The LOD1 names are not resolved (e.g. `45817e90`, `8968a63b`) | confirmed |
| sys 0x5C | 12 bytes | often ASCII `0,0.00000,0.` | unknown |
| gfx 0x00 | u32 | 1 | confirmed |
| gfx 0x04 | f32 × 3 | base (minimum delta) | confirmed |
| gfx 0x10 | f32 × 3 | step | confirmed |
| gfx 0x1C | u32 | 0 | confirmed |
| gfx 0x20 | u64 × V | delta, quantised exactly like mesh-blob positions ([§7.2](07-models.md#72-mesh-blob-vertex-buffer)): x and y 24-bit, z 16-bit | confirmed |

V is 8,983 (8,988 for `mo1`), 2,958, 1,680 or 599: LOD0 to LOD3 of the
head. The 1,626 heads have one delta for each of LODs 1–3, and 1,877
LOD0 deltas. Deltas are within ±1.5 cm.

**Vertex order.** Deltas are in one fixed vertex order per base
topology. That order is not the stored order of any head drawable,
because each head's vertex buffer is reordered on its own. Match
vertices by (UV, position) against the base head to apply them.

It is **one fixed permutation of the base mesh per class and LOD**
(confirmed, 2026-11-22), recovered from three reference heads: match each
head's offsets from the base (by UV, then nearest position) to a delta, then
give each base vertex the delta slot closest over the references. The same
permutation reproduces **1,198 of the 1,208 head drawables** at a median of
2.65 µm (the worst vertices, about 3 mm at the 95th percentile, are at UV
seams). For 158 `fr1` heads the LOD2 / LOD3 drawables are not the base plus
their LOD2 / LOD3 delta.

**Pairing heads and deltas.** Many head drawables are identical meshes
(e.g. `head_mo1_afr_002` … `_006`) with identical deltas, so heads and
deltas must be paired in order: heads in name order take the first free
matching delta in archive order. The LOD1–3 hashes (e.g. `8968a63b`,
`22f92243`, `dc6d152c` for `mr1`) take the class of the LOD0 delta before
them.

**Heads with no drawable.** 434 heads have a LOD0 delta with LOD1–3
partners but no head drawable (`mr1` 263, `fr1` 148, `mk1` 5, `mb1` 4,
`mi1` 3, `fo1` 2, `mo1` 2, `fb1` 1). 247 more LOD0 deltas have no LOD1–3
partner; 233 of them (`mr1` 146, `fr1` 87) sit in one run at
`packs.rpf@430f9010`–`43e6d010`, before any other delta. The head drawables
name only 610 `mr1` and 488 `fr1` identities against 869 and 612 delta sets,
and `mk1` and `mi1` have no head drawables at all. The base heads have only
a normal map in the install, no colour texture.

**Checks over all 6,758 resources** (a second count finds 6,757; one has hash 0):

- 100 % have the fixed system layout, a buffer of 0x20 + 8V bytes and
  zero page padding;
- deltas are in archive order of the heads (alphabetical);
- for **1,208 of 1,626 heads**, the LOD1 deltas equal (exported head −
  base head) with a median error of 2.5 µm, at the quantisation level.
  The other 418 have no exported head drawable of that class to compare
  against (more precisely, no head drawable at all; see above).

## 19.5 Other families

- **Type 5** (6,388 resources, 5,561 with `7d8f3db3`): 0.6–2 KB. They
  hold f32 boxes or capsules, such as half-extents (2.01, 0.29, 0.10)
  with mirrored minima, and bone names `BM_Head`, `PH_Glasses`,
  `HatBandTilt`, `SKEL_L/R_FOOT`, `Forehead`, `Nose`. Probably collision
  primitives for hair and cloth (inferred).
  **Box volume sets** (2026-11-23): type byte 5 at 0x1A (cloth is 2) with
  tag `7d8f3db3` at 0x1C; entries are 0x90 bytes: centre (vec3), rotation
  quaternion xyzw, half extents at +0x50 and the value 70 at +0x5C
  (meaning unknown). High: 8 sets in the minigame pack decode and their
  wireframes fit the models. A group hash map at +0x40 uses a Swiss-table
  layout (16 control bytes, then 24-byte slots; medium); what the groups
  select is unknown.
- **Type 1** (480): bone lists of whole skeletons, including animal
  tails (`SKEL_ROOT`, `SKEL_*_HAND`, `SKEL_Tail0..5`). Unknown.
- **`E` + `F` pairs** (546 + 490, 1.28 GB): an `E` system page names an
  `F` payload by a u64 key (E +0x48 = F +0x20). All 449 distinct `F`
  keys are named by an `E`. F +0x28 is the payload size in bytes, and
  the payload sits in a 0x1380 buffer.
  - `E` holds two padded counts at +0x38/+0x3C and +0x40/+0x44 (e.g.
    222/224 and 512/512) and a list of records. Some values have the
    form `offset << 8 | format`, as in the mesh-blob stream table
    (formats 0x03, 0x83).
  - Small `F` payloads (≈ 6 KB) are op-code streams.
  - Large ones (1–17 MB) start with a u32 index table (e.g. 4,288
    entries < 3,861), followed by rows of f16 coefficients that fall in
    magnitude.
  - Meaning unknown.

## 19.6 Animal material presets and the fur link

(2026-11-22.) Each animal has **one `packs.rpf` meta resource (0x7a) of
material presets**, struct `f795e8ba` (the struct the head skins use,
[§12](12-materials.md#material-presets-peds)): 260 resources, 2,636 presets
(confirmed).

| Field | Meaning | Confidence |
|---|---|---|
| `65121749` | preset name hash; a part's first preset is joaat(drawable name), the others are variations (names unknown) | confirmed |
| `b53667be` (probably `Category`) | slot: `ac82ad6b` fullbody, `ea24b45e` eyes, `378ad10c` head, `96edae5c` teeth; `bf483c65` (fur body), `343822c2` (fur head), `80c8c1af` (horn) unresolved | confirmed |
| `c4d75d79`, `f4fb3dec`, `ab5d2a85` | three u8 tint values; 255/255/255 or 0/0/0 = untinted; **palette rows of tints 0, 1, 2** on clothing items, 255 = off ([§12](12-materials.md#material-presets-peds)) | high on clothing; inferred here |
| `5ecf9331` | tint palette hash (32-bit): 7 values over 708 presets (`fcef23e3` 535, `c1713805` 99, `f12638b9` 34, `fd043742` 29, `8b21128f` 6, `d164052e` 4, `05317603` 1). `05317603` is `ped_tint_vstation_generic` ([§12](12-materials.md#material-presets-peds)); the others are in no texture store or name list | partly resolved (2026-11-23) |
| `ebacf18c[]` | parameters; the "base" one (`3eaddcd7` = `44e21c90`) holds struct `147fdab5`: `ead733fc` albedo (64-bit store hash), `d689b17f` normal, `41acf8eb` and `9d1000d8` (64-bit) | confirmed |
| `12f85293` | tag map: `1c114e61` lists the drawable (head presets); `2f9afe84` on fur presets holds the slot the fur attaches to | confirmed |

Struct `a9d57751` in the same resources is a whole-animal look listing one
preset per slot (7 on a cow: body, head, eye, horn, fur body, fur head,
teeth; inferred). Animal root fragments are a 32-triangle placeholder
plane (shader `standard_rdp`); the animal is its named parts (confirmed).
`crm_chimpanzee_western_m_01` and `_m_02` have no parts of their own and
probably use the `_m_00` meshes (inferred).

**Where a drawable's colour slot is not the colour:**

- *Missing texture:* cat eye drawables bind a hash in no store; the preset
  named after the drawable has the real albedo (512 × 512 BC7 sRGB).
  Confirmed on one cat (3 eye presets).
- *Tint mask:* cattle coats bind a linear 1024 × 1024 BC1 texture whose R
  and B are binary patch masks (0 or 255) and G greyscale shading (12–247),
  confirmed on the Holstein cow and bull; the three tint bytes probably
  colour the regions through the palette. The cow also has one untinted
  preset with a painted sRGB albedo; the bull has only tinted ones. This
  matches the deer's region mask ([§12](12-materials.md#animal-shader-ped_standard_animal)).

**Fur asset → groom pieces:**

1. On fur presets `41acf8eb` is a **64-bit asset GUID**, one per fur
   variant (confirmed).
2. A PSO asset registry (`packs.rpf@11dce0000`) holds 1,631 assets keyed by
   GUID, item struct `20e0f6ed`: GUID, `2d4958cf` asset type (30: 589,
   0: 511, 1: 200, 60: 97, 50: 95, 5: 78, 23: 59, 2: 2), `20bffba8` and
   `cb61a7e1` (64-bit, zero on type 30), properties (key → u32 list). Type
   30 is fur / hair (inferred); its list under `e3da9c8b` ends in the slot
   hash (confirmed, one registry read).
   Across all registries (confirmed): **33 PSO files** in `packs.rpf` hold
   9,342 assets of struct `20e0f6ed`; types 23: 2,580, 60: 1,747, 66: 1,639,
   30: 1,464, 1: 614, 0: 598, 50: 198, 5: 142, 15: 112. 1,853 of the piece
   table's 1,879 GUIDs are in them.
3. An unnamed resource right after the groom block (`packs.rpf@113b56010`,
   288,976 bytes) is a **hash map from GUID to a record** (confirmed): u32
   at 0x10 = `0x100` (group count), the map pointer at 0x18 and the map end
   at 0x48; 256 groups of 528 bytes from 0x60, each 16 control bytes then
   16 slots of 32 bytes {u64 GUID, 16 zero bytes, pointer}. Pointers address
   the system page followed by the graphics page as one buffer. A fur
   record is {u32 byte length, u64[length / 8]}: the fur's groom pieces.
   1,879 entries of 1–14 pieces, 6,238 distinct piece ids (e.g. cat body
   variants 8, 9, 9, 9, 9, 9, 8; cat head 4, 4, 5, 4, 4, 5, 4; cow body 1
   each, the tail switch). Other record shapes exist (not decoded). The
   archive holds 5,599 groom pieces, fewer than the ids, so some assets'
   files are not in `packs.rpf` (inferred).
4. The piece ids occur nowhere else in `packs.rpf` (searched as u64 and as
   halves), so they are probably file-name hashes resolved through the TOC.
5. **Usable link (inferred, strong evidence): grooms are stored in the
   name order of their fur asset.** Named resources are in name order
   (24,418 of 24,422 adjacent named drawables ascend, 2,827 of 2,864
   fragments). Of 234 consecutive groom chains, 220 have their roots on the
   cat's body / head mesh at a median of 0.5 mm (other chains 8 mm or
   more); the Holstein cow's chains start right after the cat block; inside
   the cat block the runs match the piece counts in alphabetical order of
   the variant.

**Checked on 11 animals** (2026-11-23; placed by this walk / of those
near the skin on their own part): cats (feral and shorthair, one shared
skin) 229 / 229; wild and legendary boar 97 / 97; chimpanzee 49 / 49; boxer
53 / 53; Holstein cow 18 / 18; white-tailed deer 130 / 127; black bear
29 / 28; coyote 44 / 41; Yorkshire terrier 132 / 118; Holstein bull 27 / 20.
Counts can be off by one (coyote `body_medium_00` lists 6 ids but has 7
files).

- **Corrections** to the first reading: the cat variants sit in exact name
  order (`body_long_00`, `body_medium_00` … `body_short_03`,
  `head_long_00`, `head_medium_00` …); the "6-chain group with longer
  strands" was the long variants (body 7 pieces, head 1), and the cat head
  extras were a LOD-grouping error. With the grouping below the heads give
  exactly the table's counts.
- **Sharing** (confirmed): variants of one animal share piece ids (the
  cow's three medium bodies list the same single id, its three medium heads
  the same five). Over 506 animal fur assets: 2,797 ids, 2,057 distinct,
  407 shared within an animal, 4 across animals (cow and runaway cow). A
  shared piece is stored once, in the run of the first variant in name
  order that lists it (inferred; the counts add up on the cat, cow, boar,
  chimpanzee and boxer).
- **Presets without a GUID** (confirmed): of 495 fur placeholder drawables,
  429 have a preset with a GUID (all in the table); 66 have none (the cat's
  long variants, legendary and corpse variants …). Their grooms exist; the
  count is known only from what the neighbours leave.
- **LOD grouping** (inferred; replaces the "same bones, fewer strands"
  chain rule above): compare bone *sets*, not order. LOD names sort apart,
  so a piece's LOD 0 can be followed by another piece's whole chain before
  its own LOD 1 (strands in archive order 571 | 827, 413, 289, 124 | 285,
  196, 74, where 571 / 285 / 196 / 74 are one piece with an identical root
  box). A groom joins one of the last three pieces when it has the same
  bones, fewer strands than that piece's last LOD and, for a piece further
  back, the same root box within 2 mm. Result: 20,010 grooms → **5,599
  pieces** (the old rule gives 5,915 chains).
- **Root-to-skin distance** (measured): 0–6 mm for most pieces on their
  own animal, 9–15 mm on the boar's long variant, 25 mm or more on other
  animals; same-sized relatives (coyote among canines) sit at 4–30 mm, so a
  threshold alone does not separate them.

### Human hair (2026-11-23)

- **Hair assets:** 911 of the type-30 assets are human hair (data
  confirmed, meanings inferred). Property `e3da9c8b` holds 10–13 tag
  hashes, many of them joaat of style words (`non`, `n`, `straight`, `m`,
  `wavy`, `sb`, `curly`, `sf`, `sm`, `l01`–`l09`, `p`, `sr`, `braided`,
  `f01`–`f03`, `full`). `3938f375` is one hash with 217 distinct values
  (two resolve to character names; one value covers 489 assets);
  `86a7e2f6` one hash with 171 values, none resolved. *Later (2026-11-23,
  confirmed):* the `e3da9c8b` list is joaat of the words of the asset's
  name (`hair`, `lob`, `wavy`, `l08`, `non`, `sf`, `n`), or the whole stem
  for a character's own asset; fit and number are not among them.
- **Piece ids** (2026-11-23): hair / fur assets list 5,591 distinct piece
  ids against 5,599 groom pieces in the archive. Nine single-piece assets
  have a piece id equal to their own GUID, so the ids look like GUIDs, not
  name hashes (inferred). The wardrobe tag `023e5a2f` often names a
  stand-in placeholder of another style (one placeholder prefix comes with
  up to 25 GUIDs). The asset → groom file link is still not stored anywhere
  found; placing by name words, archive order, piece counts and geometry
  gives a probability (780 assets placed: 86 at 0.9 or more, 330 at 0.5 or
  more).
- **No name link** (confirmed negative): of 612,380 material presets, the
  429 named after hair or fur placeholders are all animal; none of the 856
  human placeholders has a preset named after it, and no hair asset field
  is joaat of a placeholder name.
- **Archive order** (confirmed on the first 6 only): the groom sequence
  starts with 6 single-piece armpit grooms, matching the 6
  `armpithairfull_*` placeholders that sort first. Grooms share the ped
  part mesh space (head pieces at z ≈ 0.6, head meshes at z 0.42–0.71).
- **Fit to a head** (measured): 1,819 groom pieces have their roots within
  6 mm of one female head mesh, in 128 runs of neighbouring pieces; a run
  is not a style, and style boundaries inside a run are unmarked.

## Open questions

- The groom ↔ placeholder drawable link for **human** hair (`hair*`,
  `beard*`, `brows*` 3-vertex drawables): what names a hair asset's
  placeholder, which style a character wears, and where one style ends
  inside a run ([Human hair](#human-hair-2026-11-23)). For animal fur the
  order rule of [§19.6](#196-animal-material-presets-and-the-fur-link)
  links grooms to fur assets.
- Animal fur: whether the table lists a variant's ids in file order; why
  66 placeholders have no preset and where the missing piece files are;
  the other registry files and record shapes; fur as drawn (only guide
  strands are decoded; fur presets carry palette `fcef23e3` and a 16 × 16
  texture); the tint palettes and how the bytes index them. The first
  "animal" chains of the catalogue (finger bones, 1.2 m span) are probably
  human body hair.
- The meaning of cloth particle w and the second particle / edge set.
- The colour textures of the base heads (not in the install).
- Hair: A0 bytes 1–2, A6, A7, B7, Q0–Q2, C0; the twelve style values at
  system +0xF8–0x10F; placement within a triangle, clumping, noise and
  curl; the hair lighting inputs; how palette `fcef23e3` applies; how
  quality and distance scale the count.
- The meaning of hair sections not listed above; the
  tail weights; render-strand generation.
- The name of `7d8f3db3`, and of the LOD1 base-head hashes.
- The cloth per-instance structures after the first particle/edge pair.
- What the `E`/`F` pairs are (40.7 % of the unnamed bytes).

---

← [18. Radio station audio and scheduling](18-radio.md) · [Index](README.md)
