# 17. Other formats

*Part of the [SixNG format specification](README.md). Numbers as of 2026-11-21; confidence: confirmed / inferred / unknown.*

| Format | State |
|---|---|
| PSO (`PSIN`; sections LSCH/LENM/LSNS/LGID/LRSC/CHKS) | **decoded** (2026-11-21): the plain-file form of the meta format of [§8](08-meta.md). Section headers are big-endian; the block table and references are little-endian; the `LSCH` schema names fields with 32-bit joaat (struct names are 64-bit). 2,225 of 2,281 files in `data.rpf` decode. See [PSO](#pso) and [Vehicle handling](#vehicle-handling) |
| RBF (`RBF0`) | a new variant with u16 indices and inline names; exported raw |
| `*_bvh4.rpf` | dehydrated GPU acceleration structures and a mount manifest. See [GPU BVH4](#gpu-bvh4) |
| XML, text | plain; exported as is |
| shaders (`shaders.rpf`) | compiled cache and DXBC-wrapped native shader programs. See [structure/06](../structure/06-models-shaders.md) |
| audio (AWC/ADAT) | container layout, chunk tags and Opus framing in [structure/03](../structure/03-audio-awc.md); radio stations and scheduling in [§18](18-radio.md); exported by `sixng audio-export` ([tool](../tool/02-cli.md#audio)) |
| UI fonts | Scaleform font libraries, DefineFont3 outlines (confirmed). See [Scaleform UI fonts](#scaleform-ui-fonts) |
| particle effects | dictionaries, rule names, keyframe curves; rule records open. See [Particle effect dictionaries](#particle-effect-dictionaries) |
| animation (`anim.rpf`) | clip dictionaries and cutscene metadata; clip metadata decodes (confirmed); skeleton not in the clip dictionaries. See [Animation](#animation) |
| ped body shape | presets in `common.rpf`, a control-to-input PSO and pose rigs in `anim.rpf`; the rig program runs (confirmed). See [Ped body shape](#ped-body-shape) |
| music catalogue (`audio:/config/music.dat`) | title, artist and playlist records; title hash solved (confirmed). See [Music catalogue](#music-catalogue) |
| PSIN scene placements (`story000_script.rpf`) | 97,504 64-byte placement records (confirmed). See [PSIN scene placements](#psin-scene-placements) |
| movies (`movies.rpf`) | Bink 2: 515 movies, 505 `KB2n` and 10 `KB2j`, stored uncompressed; header, frame index and Bink Audio tracks parse (confirmed); all 515 decode with a public Bink 2 decoder (high). 276 more in three content packs. See [Video](#video) |
| UI graphics | named texture dictionaries in `data.rpf` and UI texture stores. See [UI graphics](#ui-graphics) |
| world height map (`MHIU`, `common.rpf`) | 2048 × 2048 u8 surface heights in row runs, two copies (high). See [World height map](#world-height-map-mhiu) |
| text (GXT) | `0DRP` type `0x7a` key/text resources; 54,118 keys (confirmed). All 14,482 tables of `data.rpf` (4.21 M strings, one table per language) and the 78 of `shared_minigame000` decode (confirmed). See [Text (GXT)](#text-gxt) |

## PSO

PSO files are the plain-file form of the self-describing meta format
([§8](08-meta.md)). Each section is a 4-byte magic and a **big-endian** u32
size that includes the 8-byte header
([structure/04](../structure/04-pso-objectgraph.md)). Everything inside is
little-endian.

| Section | Contents | Confidence |
|---|---|---|
| `PSIN` | instance data: a block table followed by 16-byte-aligned blocks. See [the v22 and unversioned script layouts](../structure/04-pso-objectgraph.md#script-psin-headers). References are u64 `(offset << 20) \| block` with a 1-based block | confirmed |
| `LSCH` | schema: 32-byte struct entries {u64 name hash, u32 info, u32 0, 8 × `0xAA`, u32 size, u16 0, u16 member count}, then all members in table order as 16-byte records **identical to §8**: {u32 name, u32 offset, u8 type, u8 sub, u16 reference, u32 extra} | confirmed |
| `LRSC` | a copy of the meta resource header (`0DRP`, type 0x7a): +0x38 u16 struct count, +0x3A u16 root block, +0x3C u16 block count | confirmed |
| `LENM` | enums, 64-bit names | not decoded |
| `LSNS` | [table framing](../structure/04-pso-objectgraph.md#lsns-framing) and 16-byte records in the script bundles | confirmed framing; record contents unresolved |
| `LGID`, `CHKS` | — | not decoded |

- **Names**: struct names are 64-bit hashes (function unknown). Member
  names are 32-bit **case-sensitive joaat**, and GTA V's names match
  (`fMass`, `fInitialDragCoeff`, `fGearRatios`, `fSteeringLock`). A struct
  member's `extra` is the low 32 bits of the struct name (confirmed).
- **info**: 0x200 / 0x300 / 0x400 = alignment 4 / 8 / 16; low byte 0x0C or
  0x24 marks generated helper types such as hash-table storage, named
  `<hash32>_000001xx` (inferred).
- **Pointers** are always pool references. There are no `0x5xxxxxxx`
  resource pointers; a reference whose offset is 0x500–0x5FF looks like
  one, so do not test for them (confirmed).
- In `data.rpf`, 2,225 of 2,281 PSO files decode this way. The other 56 use
  the older GTA V layout (`PSIN` / `PMAP` / `PSCH`) and are not decoded.

Type codes not in [§8](08-meta.md):

| Code | Type | Confidence |
|---|---|---|
| 0x01 | bool (1 B) | confirmed |
| 0x4d | 64-bit hash or id (8 B) | confirmed |
| 0x50 | inline fixed array: `extra` elements of member `reference` | confirmed |
| 0x75 | map {u32 count, u32 capacity, u64 ref} to a hash-table struct: control bytes `b8794c89` (u8 × N; 0x80 = empty, else 7 hash bits, as in a Swiss table) and slots `93c495af` {`6098a50e` key, `063fa3f2` value} | confirmed |
| 0x62 | tagged value (16 B: u32 tag, then the payload) | inferred |
| 0x31 | 4 floats | inferred |

In PSO, `0x65` bit sets take 16 bytes (the first u64 is used) and `0x44`
takes 16 bytes, probably a string reference into `LSNS` (not decoded).

## GPU BVH4

The sampled `core_bvh4.rpf` and shared minigame BVH4 entries carry
dehydrated GPU ray-tracing acceleration structures. CPU collision
bounds have their own [phBound layout](13-collision.md).

After inflating an entry, the 0xD0-byte system block gives the
graphics size as a u32 at `+0xA0`. The graphics block has a
0x40-byte header at `+0x100`. The u32 at graphics `+0x104` counts
64-byte units. The size check is
`graphics_size == 0x140 + 64 * units`. This held in both sampled
archive families. Observed hierarchy words carry a 24-bit value in
bits 0–23 and a child count of 0–4 in bits 24–31.

In the minigame pack (2026-11-23) the version word is `06000281` and a
bounding box sits at `+0x110` / `+0x11C`. The node data is a private
hardware layout (`0x81` fill, big-endian child counts 2–4) and is not
decoded.

**No stored geometry** (2026-11-23; high): across all 1,241 files of the
minigame BVH4 archive, 90.7 % of the bytes are zero and only 0.15 % of the
4-byte words read as floats inside the file's own bounding box, so there
are no stored boxes or triangles; the runtime builder fills the buffer from
the model. The node blocks (every 0x2000 bytes from a start that varies per
file and is not in the header: 64 big-endian child counts, then
little-endian child ids = byte offset >> 3) stay unexplained (low).

`story000_bvh4.rpf` contains four zlib-compressed XML records.
Its content and setup records reference terrain and merged BVH packs
for mounting. Joining archived BVH4s back to meshes and rehydrating
the console representation remain open.

## Vehicle handling

The PSO at `data.rpf@0x1c35b000` holds 1,097 handling rows of struct
`1ddb46e190779a14` (72 bytes per row, PSIN block tag `0x90779a14`).

| Field | Row offset | Meaning |
|---|---|---|
| `93db5464` | `+0x08` | Handling name hash (lower-case joaat, e.g. `phoenix`); 301 match model names |
| `49a01a86` | `+0x0C` | `ParentName`: second name hash; all 624 nonzero values resolve to another handling row by `93db5464` |
| `41c8dec7` | `+0x10` | `HandlingData`: `{u32 count, u32 capacity, u64 pool_ref}` to an array of u64 pool references |

Each array element addresses a component instance; use the target PSIN
block's struct tag to decode it. All 3,481 references across 1,034 nonempty
arrays resolve to aligned records of 60 distinct struct tags. The second
name hash is consistent with a base-profile link; runtime inheritance or
component override rules are not verified.

Join [vehicle model info](11-vehicle-paint.md#vehicle-tables) `handlingId`
(`ec41bd9c`, `+0xAC`) to `93db5464`, not `gameName`. All 327 model-info rows
in the merged table at `packs.rpf@0xa934010` resolve uniquely; 48 store
`gameName != handlingId`. For example, model-info row `0x11523b0` of that
inflated resource stores `gameName=0xe54c8bcf` and `handlingId=0x32e37f93`
(`airgator`).

| Component | Contents (inferred from the field names) |
|---|---|
| `b4e0f8b0` | body: `fMass`, `fInitialDragCoeff`, 3-vectors |
| `6fdd2f18` | transmission: `fGearRatios`, `fFinalDriveRatio`, `fDriveBias*` |
| `88d3a1a3` | engine: RPM limits (e.g. 5,800), a torque curve as a map of {rpm, value} records, `fOilVolume` |
| `70317f27` | fuel |
| `edf48412` | wheels, steering and brakes: `fSteeringLock`, `fBrakeForce`, `fWheelMass` |
| `bdaf2dc0` | suspension: travel, damping |
| `c746f5ed`, `82173b9e`, `0da11ed9` | unknown |

**Steering lock** (2026-11-23; field and join high, inheritance medium):
`fSteeringLock` (`d9cd734b`) in the wheels component is in radians: 0.72
(41.3°) on a sports car, 42.7° on a saloon, 36.4° on an ambulance, 51° on a
dirt bike; most cars are at 39–41°. A record without the field takes it from
its base record (`HANDLING_BASE`) through the `ParentName` chain; with that,
333 of 373 vehicle models resolve to a lock.

Mass, drag, gear ratios, final drive, brakes, steering lock, wheel mass,
suspension travel and damping, anti-roll bar force and fuel are named; most
other fields, the torque curve's included, are decoded values with hashed
names. The record's root block is a small map of two keys to 64-bit ids.

Another PSO holds lights and sirens (`Lights`, `Sirens`: cone angles,
intensity, flash timings), like GTA V's `carcols`. The paint colours are in
the game-data meta resource instead ([§11](11-vehicle-paint.md#vehicle-tables)).

## Video

`movies.rpf` has a plain payload (only the table of contents is encrypted,
as everywhere), and its videos are **stored uncompressed**, so a search for
zlib streams finds nothing in it (confirmed).

Each video is a standard Bink 2 file starting on a 4 KiB boundary; the next
starts at the first 4 KiB boundary after the previous one ends. That gives
515 files, the entry count in the archive header, covering 8,831,336,892 of
the archive's 8,832,481,776 bytes: 505 `KB2n` and 10 `KB2j`, about 236
minutes, all at 30 fps (confirmed).

Header (the public Bink container layout; all u32):

| Off | Field |
|---|---|
| 0 | signature (`KB2n`, `KB2j`) |
| 4 | file size − 8 |
| 8 | frame count |
| 12 | largest frame size |
| 16 | frame count again |
| 20, 24 | width, height |
| 28, 32 | fps numerator, denominator |
| 36 | video flags (`0x40013` typical); **bit `0x100000` = transparency channel** (`0x140012` / `0x140013`; 2026-11-23, high, see below) |
| 40 | audio track count n |
| 44 | Bink 2 only: one more u32 (`0005e987` seen) |
| then | n × u32 largest decoded audio size, n × (u16 sample rate, u16 flags; `0x7000` on 48 kHz stereo), n × u32 track id |
| then | (frame count + 1) × u32 frame offsets: bit 0 = keyframe, the last one is the file size |

Every frame table checks out against the file size.

| Resolution | Files |
|---|---|
| 1280 × 720 | 246 |
| 1920 × 1080 | 130 |
| 720 × 1280 (portrait) | 67 |
| 512 × 512 | 23 |
| 1024 × 1024 | 19 |
| 2560 × 1440 | 14 |
| 512 × 256, 2048 × 558 | 2 each |

Audio: 319 files have none, 165 one track, 30 two, 1 four; all 48 kHz
stereo. The longest video runs 534 s. Original file names are not stored
with the files. Matching them in order against a plain-text content list
is reported to recover names for some groups (unverified: the list is
not identified).

### Decoding

(2026-11-23, high confidence.) A recent open-source Bink 2 video decoder
(with Bink Audio RDFT / DCT) decodes **all 515 files** with no errors,
`KB2n` and `KB2j`, including the 246 at 1280 × 720 and the 67 at
720 × 1280, carved by the header above. Open: there are reports of
artifacts at 720p in that decoder; the 720p clips look clean on contact
sheets (6 frames each) but were not checked frame by frame.

### Bink files in other archives

(2026-11-23, high.) A raw-byte scan of every other archive under `pack/` and
`data/` (78 files; `KB2` + a lower-case letter at any offset with a
plausible header) finds Bink 2 only in three content packs, all `KB2n`, on
4 KiB boundaries as in `movies.rpf`:

| Archive | Files | Content |
|---|---|---|
| `pack/core_restofworld` | 5 | in-world TV shows |
| `pack/story000/story000.rpf` | 264 | story-mission screen content: CCTV, phones, PCs, news, logo animations, in-world TV and console screens showing the minigames |
| `pack/story000_restofworld` | 7 | social media and news clips |

That makes **791 Bink files** with `movies.rpf`; `shared_minigame000` has
none. Resolutions in the two story packs: 720 × 1280 101, 1920 × 1080 58,
1280 × 720 31, 2560 × 1440 22, 540 × 960 14, 1000 × 600 7, then 1024 ×
1024, 1000 × 750 and 1000 × 500 (4 each) and 960 × 720 (3). Open: a Bink
inside a compressed or encrypted entry would be missed; Bink 1 was not
searched for outside `movies.rpf`.

### Minigame movies in `movies.rpf`

(2026-11-23; identification high, descriptions medium; names are
descriptive, the original names are unknown.) Only the first 24 files in
archive order belong to the arcade minigames, and they include **all 14
files at 2560 × 1440**. `(width, height, frames)` is unique per file
except 020 / 021, so it works as a lookup key. All at 30 fps.

| # | Game | Content | Size, frames | Audio |
|---|---|---|---|---|
| 000–002 | hunting game | title loop, intro, attract | 2560 × 1440; 115, 402, 448 | no, yes, yes |
| 003 | probably the hunting game | cabin interior / exterior | 1920 × 1080, 478 | no |
| 004 | console | console boot logo | 1920 × 1080, 198 | yes |
| 005 | beat-'em-up | subway loading screen | 2560 × 1440, 255 | no |
| 006, 011–014, 017, 018 | time shooter | install, boot, title, intro, cabinet screen, emergency landing, time travel | 2560 × 1440; 799, 372, 123, 733, 302, 509, 602 | yes |
| 007–010, 015, 016 | probably the time shooter | screen-edge glow overlays, about 3 s each; **007–010 and 015 carry a transparency channel** | 1920 × 1080; 85 or 86 frames (016: 91) | no |
| 019, 022, 023 | puzzle game | studio logos, intro, title | 2560 × 1440; 601, 2177, 297 | yes, yes, no |
| 020, 021 | puzzle game | credits, and a white version of the same size (a mask or variant) | 1920 × 1080, 3,382 | no |

**Transparency channel** (2026-11-23; high, 24 movies inspected): five of
the six overlays (007–010, 015) have video flags `0x140012` or `0x140013`;
016 and the ordinary cutscenes have `0x40012` / `0x40013`. A Bink 2 decoder
outputs the five as YUV 4:2:0 with alpha and the others without, and the
channel holds a sensible edge-glow mask, so flag bit `0x100000` marks a
transparency channel. On four of the five the mask is exactly half in the
screen centre. Open: whether colour is premultiplied, how the game blends
them, and which game event each belongs to.

The rest of the archive is in-world TV adverts, phone and social clips,
effect and UI loops, and the publisher logo (177). **The beat-'em-up's long
intro is not a Bink file** in any scanned archive, so it is almost
certainly rendered in-engine (medium-high); the story pack holds its
footage only as in-world screens (store page, gameplay, title menu).

### Bink Audio framing

At the start of each indexed frame, read one `u32 packet_size` and
its payload per audio track, in track order. A nonempty payload begins
with a `u32 decoded_byte_count` followed by the encoded audio. Bound
every packet by the next frame offset or the declared file size.

The encoded audio after `decoded_byte_count` is Bink Audio. The sum of a
track's `decoded_byte_count` values is its 16-bit PCM size, so samples per
channel = sum / (2 × channels); the track durations computed this way
match the video length within one frame across the audio-bearing files.

## UI graphics

- **Named texture dictionaries in `data.rpf`** (75): 36 loading screens,
  `hud`, `blip`, `map_scripted_movie`, `generic_movie`, `credits`, `phone`,
  `controller_test`, `analog_instruments`, `generic_instruments` and 25
  `vit_*` vehicle gauge sets (dashboards, motorcycle displays). Their 880
  textures decode with the dictionary rules of [§3](03-texture-dictionaries.md)
  and [§4](04-xbox-tiling.md); texture names are placeholders (`hud_i8d3`).
- **UI texture store** `data/ttdb/ui/ttdb_ui.ttpf` ([§5](05-ttpf.md)): 10,013
  textures, one mip each, named only by 64-bit hash. Mostly sprite atlases:
  shop and brand logos, tattoo art, hair, beard and wheel icons, character
  portraits, in-game arcade and website art. Formats: BC1 3,801, BC7 3,383
  plus 1,828 sRGB, BC3 928, BC4 57, A8 16. With BC1 BCPack decoded
  ([§6](06-bcpack.md)), 9,012 decode completely, 345 partly (some 64 KiB
  chunks are BCPack in another format: the top of the image decodes, the
  rest is blank) and 656 lie only in such chunks, mostly BC7.
- **Fonts and HUD shapes**: `data/ttdb/graphics` holds single-channel font
  glyph atlases and HUD shapes. The UI fonts themselves are vector outlines
  in Scaleform font libraries ([below](#scaleform-ui-fonts)).
- **Arcade console** (2026-11-23; what it is high, names medium): three
  2048 × 2048 BC textures in the UI store are whole 16:9 console home pages
  squeezed to square (top bar, logo, description, key art, a row of three
  tiles), one per game: `608bf472bbebdf5a`, `50b4807ac9c04456`,
  `28fccff5c16fee70`. Their text sits about a tenth of the screen lower
  than in the story movies showing the console, so they may be a store-page
  variant. Those movies also show three titles with no data found in the
  UI store. Crosshairs `e4c7da5c5cb2476c` (thin four-arc ring) and
  `d82d7baf2177b8fa` (thicker) are not the hunting game's reticle as seen
  in footage, which the game composes or replaces (medium).
- **Minimap / pause map**: no tile set found in these stores or
  dictionaries. The map may be drawn from world data at runtime (not
  established). Menu layouts (which sprite goes where) are not decoded.

### Scaleform UI fonts

(Confirmed, 2026-11-22: all 29 fonts load and render.) The font libraries
are GFX movies in `data.rpf`: `fontlib_efigs` (`@adf3000`, 17 fonts) and
the Chinese, Japanese and Korean libraries at `@a3ff000`, `@a983000`,
`@ae83000`, `@b2f2000` and `@b761000`.

- Header: `GFX` (plain) or `CFX` (zlib after 8 bytes), a version byte, u32
  length; then a RECT (nbits = first byte >> 3, size (5 + 4·nbits + 7) / 8
  bytes), u16 frame rate, u16 frame count, then SWF tags (u16 `code << 6 |
  len`; len 0x3F means a u32 length follows). Custom tags (code ≥ 1000) sit
  among them.
- **Tag 75, DefineFont3:** u16 id; u8 flags (0x80 has layout, 0x08 wide
  offsets, 0x04 wide codes, 0x02 italic, 0x01 bold); u8 language; u8 name
  length and name; u16 glyph count; an offset table (u16 or u32, count + 1
  entries, the last being the code-table offset); one SHAPE per glyph (4-bit
  fill and line bits, then edge and style records); the code table (u16
  UCS-2); then the layout: ascent u16, descent u16, leading i16, advances
  i16[n], bounds and kerning.
- Units: an EM of 1024 × 20 twips, y down.

Several fonts are subsets of a full font (digits only, a few icons). UI
textures are referenced by a 64-bit `ttpf` hash with an unknown function,
so they cannot be looked up by name; HUDs are probably drawn by script.

## Particle effect dictionaries

(2026-11-22.) Unnamed resources with no geometry and at least one embedded
texture named `keyframeTexture` (R16G16B16A16_FLOAT, 16 × 4, 16 samples in
4 rows). It was first read as property curves over a particle's life; later
decoding says it is **not** a set of life curves (the life curves are the
keyframe properties below), so what it holds is open. `data.rpf` has 146.

- Rule names are plain strings in the system page. The dictionary name is
  the string that prefixes the most others with `_` (`scr_arc_tl` →
  `scr_arc_tl_candle_flame_flare`). A dictionary can also hold rules of
  shared sets (`MG3_blood_…`).
- Arcade sets: `scr_arc_bb` (`@f75010`, blood), `scr_arc_st` (`@f7a010`) and
  `ent_anm_arc_st` (`@b13010`), `scr_arc_svr` (`@f7e010`),
  `weap_arc_tj_tracer` (`@f85010`, tracers), `scr_arc_tl` (`@f89010`,
  candle flame, shadowy figure).
- Open: the rule records (emitter parameters) and the sprite references.
  The sprite sheets are not embedded, and none of the 344,561 `ttpf` hashes
  of the 57 stores occurs in the dictionaries. *(Later decoding resolves
  particle sprites to the effects texture store and reports sprite slots
  per rule, below, so the earlier search probably missed them; not
  reconciled in detail.)*

### Rule tables

(Offsets measured on the arcade dictionaries; property names confirmed by
hash; units unverified.)

- **Root** (system page +0): +0x10 dictionary name; +0x20 texture /
  drawable table (unused); +0x28 effect rules, +0x30 emitter rules, +0x38
  particle rules. Each table has a hash array at +0x20 (joaat of the rule
  name), u32 count | capacity << 16 at +0x28 and a pointer array at +0x30.
- **Effect rule** (0x240 B): +0x18 f32 (4.2 on every arcade effect), +0x20
  name, +0x38 event array (u16 count at +0x40; an event has its emitter rule
  at +0x50 and particle rule at +0x58), +0x48 evolution list ("LOD",
  "intensity"), floats at +0x98..+0xB8, and keyframe properties at
  +0xB8 + 0x40·i: colourTintMin, colourTintMax, zoomScalar, dataSphere,
  dataCapsule.
- **Emitter rule** (0x300 B): +0x18 f32, +0x1C own hash, +0x20 name; ten
  keyframe properties at +0x40 + 0x40·i, the same in every emitter:
  spawnRateOverTime, spawnRateOverDist, particleLife, playbackRateScalar,
  speedScalar, sizeScalar, accnScalar, dampeningScalar,
  matrixWeightScalar, inheritVelocity.
- **Keyframe property** (0x40 B): +0x10 key list, +0x18 u16 count | u16
  capacity, +0x30 id hash (stored twice). **Key** (0x20 B): time 0..1 at +0,
  1 / (time − previous time) at +4, value vec4 at +0x10 (min / max in x / y
  for ranged properties).
- **Property id = joaat(lower("`<class>:m_<member>KFP`"))**. Behaviour
  properties resolve for ptxu_Acceleration, Size, Rotation, Dampening,
  MatrixWeight, ZCull, AnimateTexture, Decal, Light, Attractor, FogVolume
  and ptxd_Trail; ptxu_Velocity:m_angleKFP = `2abaae94`. ptxu_Colour has
  emissiveIntensity plus five unresolved ids (`e7851995`, `c1444d14`,
  `6dfba680`, `a0930bb2`, `d24def27`).
- **Particle rule** (0x430 B): +0x178 name, +0x188 behaviour pointer list
  (u16 count at +0x190; behaviours carry keyframe properties at +0x40·k,
  their type hashes resolve to no known name), +0x388 keyframe texture
  (null on about 7 %). +0x248..+0x2E8: four {pointer, short half-float
  string, u32 pair} records (shader variables?); +0x300..+0x360 sparse
  64-bit values, probably the sprite references; +0x370 a list (u16 count
  at +0x378) of shared hashes, probably shader / technique ids (guesses).

**Which game each arcade set serves** (2026-11-23; from effect names; high
for four, medium for `scr_arc_st`): `scr_arc_bb` = the hunting game,
`scr_arc_svr` = the zombie survivor shooter, `scr_arc_tl` = the first-person
puzzle game, `weap_arc_tj_tracer` = the twin-stick time shooter (its muzzle
flashes `weap_arc_tj_muz_rifle` / `_handgun` / `_shotgun` and
`weap_arc_tj_tracer_shotgun` sit in the big weapons dictionary, 226
effects), `scr_arc_st` = probably the beat-'em-up. The pack's streaming
tables name five arcade games: `arcade_survivor`, `arcade_twinstick`,
`arcade_beatemup`, `arcade_hunter`, `arcade_indiepuzzle`. The minigame pack
holds no ped or skeleton resources.

**Rule properties, read by rebuilding effects** (2026-11-23; medium):
`particleLife` is seconds (blood blobs 2–3 s, mist 0.6–0.9 s, tracers
0.01–0.05 s); `speedScalar` m/s; `sizeScalar` a percentage applied to
`ptxu_Size`; `ptxu_Acceleration` z is up (gravity z = −9.8); colours look
linear. The unnamed `ptxu_Colour` properties `e7851995` and `c1444d14` are
the colour-over-life min / max; a rule whose alpha keys are all 0 still
draws. Some rules carry two `ptxu_Acceleration` min / max pairs (which
applies is open). Trail rules (`ptxd_Trail`) keep the width in the height
of the size keys: the hunting game's tracer is a streak 0.03 wide tapering
to 0 plus a smoke trail 0.66 wide.

Sprite slots by format and how they render (about 60 rules; medium):
`0x310` and `0x338` BC4 shape mask (red); `0x318` BC6H colour sprite;
`0x350` BC7 sRGB; `0x340` and `0x348` BC5 (normal or lighting); `0x300`
and `0x308` BC7 atlases. Atlases are either flipbooks (e.g. 2048 × 1024 with
32 frames = 8 × 4) or variant sheets with no animation, one cell per
particle (2 × 2 splatter `61f740590c748d11`, splash `d59d86542ed1a0c0`,
droplets `b3daa343c4f83c93`). Open: effect duration (rules give spawn rates
only), emission direction and cone, the keyframe textures, the other
unnamed colour properties.

## Audio game-data containers

(2026-11-22.) The audio game-data bins of the minigame pack: u32 version,
u32 data length, the data, u32 name-table length and bytes, u32 length and
bytes, u32 count, then count × {u32 name hash, u32 offset into the data}.
The pack's eight bins are, in archive order, `audioconfig` (64 objects),
`curves` (65), `dynamix` (80), `game` (81), `modularsynth` (7,193), `music`
(9 graph objects), `sounds` (4,436) and `speech` (616). Music objects do
not reference streams by hash; their node bytecode is not decoded. The
pack's PSOs are dialogue, zones, lights and fog, and its 787 meta resources
are map data: no minigame tuning tables are in data files.

### Audio object bodies

(2026-11-23.) The data block has `0x045a88ea` at +8; after it come two
tables of {u32 length, count, …} and the name index. **Each object's first
byte is its class.**

- **Sound file (schema 166).** Classes follow the usual numbering: 12
  simple, 6 wrapper, 14 randomized, 7 sequential. **References between
  objects are u32 indices into the objects in data-offset order** (high: 16
  of 17 multi-child groups share a bank in that order, against 1 of 19 in
  hash order).
  - Simple sound body: container hash (the bank), wave hash (**its low 29
    bits are the stream id in the bank**), u8, u8, u8 n, n references
    (high: 2,050 of 2,051 match a real stream; the trailing references'
    role is unknown).
  - Randomized, sequential and class-8 sounds end with u8 n, n child
    references. A wrapper's last 12 bytes are child, u32, reference.
  - **Child references of more classes** (2026-11-23, re-measured on the
    main sound file; offsets from the start of the class body, which begins
    where the header ends; `0xffffffff` = none). A fixed offset counts when
    at least 96 % of its values are valid indices and they lie nearer the
    parent than chance:

    | Class | Child references | Confidence |
    |---|---|---|
    | 15 | 6 bytes, u8 k, k bytes, u8 n, then n × (u32 child, f32 mostly 1.0, u8) | high (70 of 70) |
    | 24 | a count byte, then n references at the end of the body | high (2,866 of 2,883 fit, none bad) |
    | 1 | at 6 | high |
    | 2 | at 56 | high |
    | 3 | at 29 and 33 | high |
    | 5 | at 0, 4 and 8 | high |
    | 9 | at 20 | high |
    | 10 | at 0 and 4 | high |
    | 19 | at 33 | high |
    | 20, 27, 29, 30, 31, 36, 40, 48 | at 0 | high |
    | 28 | at 0 and 4 | high |
    | 41 | at 0, 6 and 12 | high |
    | 46 | at 9 | high |

    *Retracted (2026-11-23):* an earlier, smaller sample also listed class
    16 at 15, class 30 at 25 and class 48 at 8. On the full file those are
    not references: class 16 at 15 is a bit mask (`0x0e`, `0x1e` …
    `0x1fe`) and the other two are constants. With these references about half of the main file's
    sounds reach a simple sound (the earlier sample gave 2,228 of 2,385
    non-simple sounds).
  - **Class 16 body** (2026-11-23, high: 8,248 of 9,093 objects fit): u8 0,
    u8 k, three u32 name hashes (matching no object), u8 bit count m, u32
    mask with m bits set from bit 1 (`0x0e`, `0x1e` … `0xffe`), u32 value,
    zero padding, u8 2, u8 c = floor(m / 2), then c one-byte slots. All
    26,329 slots are `0xff` (empty), so the file holds no child links for
    this class; they are probably filled at run time (inferred). Open: k
    and the three hashes.
  - **The main sound file is split** (2026-11-23, high): its 241,999
    objects are carved as two entries. Between them, past the object data
    and inside the file's second hash table, sit three 64 KiB chunks that
    are not zlib (entropy 8.0 bits per byte). Only that table's length is
    needed: joining the two pieces and zero-filling the gap parses the
    whole file. What the chunks hold is unknown. The same pattern occurs
    in a schema-18 file in `story000`: 52 stored chunks (3.25 MB) inside
    its second table, after the objects (high).
- **Game objects: classes 50–77** (2026-11-23, high). They have no sound
  header: the class byte is followed by 4 bytes of 2-bit flags (`0xaa` =
  all unset), then the body. Class 54 starts with its own name hash (2,557
  of 2,557). Links to sounds (data-order indices as above, no bad
  references; 1,928 links in the main file):

  | Class | Sound references | Fit |
  |---|---|---|
  | 51 | u16 count, then the references | 130 of 130 |
  | 52 | u16, u16 count, then pairs (reference, reference or `0xffffffff`) | 87 of 87 |
  | 56 | at body offsets 6 and 10 | 6 of 6 |
  | 61, 62, 64 | at 0 | high |
  | 68 | at 14 | high |

  Their other fields are unknown.
- **Sound header** (every class except 17, 49, 56 and 57; high: 4,392 of
  4,395 headers decode, each simple sound's container starts where its
  header ends, 2,051 of 2,051, and 2,015 of 2,016 wrappers then have
  exactly 12 body bytes): byte 1 is a mask, each set bit adds one sub-mask
  byte, and each sub-mask bit adds a fixed-size field. (1,0) is 8 bytes of
  2-bit tristate flags; (0,0) and (0,2) are nested u8 masks whose 2-byte
  values come after all other header fields.

  *Corrected field sizes* (2026-11-23, high; field (mask bit, sub-mask
  bit); replaces the earlier list, which had (0,1) as 2 bytes and covered
  fewer fields; with these sizes fixed-size classes get a single body
  length on 97 % of 112 K objects):

  | Size | Fields |
  |---|---|
  | 8 bytes | (1,0) |
  | 4 bytes | (1,1), (2,4), (2,5), (3,2), (4,0), (4,5), (5,1)–(5,4), (5,6), (5,7), (6,0)–(6,3), (6,5), (7,1), (7,4)–(7,6) |
  | 3 bytes | (0,1), (0,3) |
  | 2 bytes | (1,2)–(1,6), (2,0)–(2,3), (2,6), (2,7), (3,0), (3,1), (3,3)–(3,6), (4,1) |
  | 1 byte | (3,7), (4,2), (4,6), (4,7), (5,0) |
  | nested | (0,0), (0,2): a mask byte, then 2 bytes per set bit at the end |
  | unknown | (0,5), (0,7), (1,7), (4,3), (4,4), (5,5), (6,7), (7,0), (7,2), (7,3), (7,7); seen mostly in classes 51–77, which are not sound classes; since those game objects have no sound header (below), these may be artefacts of reading them as one (inferred) |
- Open: field meanings ((1,2) looks like a volume in mB), the other body
  fields, the unknown header field sizes, the gap chunks of the split
  file, the headerless classes 17, 49 and 57, the game-object fields, and the other schemas
  (game objects, speech and others use a different header).
- **Vehicle game objects** (2026-11-23): the game-object file (schema 581,
  115,183 objects) has one object per vehicle, named by the model name hash
  (the same joaat as the model): class 9, 108 bytes. After a 12-byte start
  (class, a flags word, the name hash again) come about 17 u32 references
  to other objects of the same file, as indices in data-offset order like
  the sound files, `0xffffffff` = none. The targets are of classes 206 (24
  bytes), 209 (40), 238 (28), 218 (8), 252 (12), 161 (36) and 44 (200);
  several are shared by all vehicles (the same class-44 objects at the same
  positions in three vehicles). High for the object and the reference
  convention; no field meaning is known (which reference is the horn, for
  example).
  *Re-checked on six records (2026-11-23, medium):* not every u32 is a
  reference. Each is `0xffffffff`, a small number (`0x22`, `0x7c7`–`0x7cb`;
  the file's first table has 2,040 entries, so probably an index into it),
  or an object reference. Emergency vehicles fill fewer fields than cars.
  No sound name hash appears in the record, so which vehicle uses which
  siren or horn is still unknown.
  *File tables (2026-11-24, medium):* after the hash index comes a table of
  one 12-byte entry per object (8 bytes of unknown meaning, then the
  object's data offset; every offset matches an object). Then comes a
  sorted list of 255,535 data positions; in the first 20,000, about three
  quarters hold an object index or `0xffffffff`. The first table after the
  data groups 7,305 hashes into 2,040 lists (the 2,040 entries above).
  249 of 315 vehicle models have a class 9 record. No game object holds the
  hash of a known siren sound or of a sound above one, so where a vehicle's
  siren is chosen is still unknown.
- **Siren sounds** (2026-11-23; names and waves high, from decoded audio):
  the sound file holds class-12 simple sounds under plain joaat names:
  `RESIDENT_VEHICLES_SIREN_WAIL_01`–`_04`,
  `RESIDENT_VEHICLES_SIREN_QUICK_01`–`_04`,
  `RESIDENT_VEHICLES_SIREN_FIRETRUCK_WAIL_01`,
  `RESIDENT_VEHICLES_SIREN_FIRETRUCK_QUICK_01`, `VEHICLES_HORNS_SIREN_1`
  and `_2` (the same waves as `WAIL_01` and `QUICK_01`),
  `VEHICLES_HORNS_AMBULANCE_WARNING` and `SIRENS_AIRHORN`;
  `VEHICLES_HORNS_POLICE_WARNING` and `VEHICLES_HORNS_FIRETRUCK_WARNING`
  are class-6 wrappers with one child. The category object
  `VEHICLES_SIRENS` is in the schema-41 file. All their waves are in one
  container (name hash `af365ada`), a bank of 58 streams: 26 sounds use 24
  of them, 12 of those sounds unnamed. Wails last 3.8–6.3 s and yelps
  0.25–0.5 s, mono 48 kHz Opus. All ten named simple sounds confirm the
  stream id rule above (low 29 bits of the wave hash). Open: the horn
  sounds themselves.

## Animation

**Convertible roof clips** are not in `anim.rpf`: see
[below](#convertible-roof-clips).

`anim.rpf` holds two families. Bodies in the low region of the archive are
`0DRP` v122 cutscene/scenario metadata; above them, bodies whose
`+0x10..+0x3F` is all zero are clip dictionaries (inferred from a sample of
seven windows).

Clip names, durations and sequence channels decode (confirmed). Sampled rotation
tracks validate as unit quaternions: 3.95 M values, worst |q| − 1 about 1.2e-7
(confirmed). A unit norm alone does not validate the decoded pose. The
skeleton is not stored in the clip dictionaries.

### Dictionary clip index

In the sampled Xbox Series 1.0.2.1 dictionaries, body `+0x38` points to
the dictionary name and `+0x68` stores its lowercase joaat. Body `+0x48`
points to `u64` bucket heads, with `u32` bucket count at `+0x50` and
`u32` total entry count at `+0x58`. A zero pointer marks an empty bucket
or the end of a chain.

| Entry offset | Type | Meaning |
|---|---|---|
| `+0x00` | `u32` | Clip stem hash, matching clip record `+0x48` |
| `+0x04` | `u32` | Clip set hash, matching clip record `+0x4c` |
| `+0x08` | ptr | Points 0x10 bytes before the clip record base used below |
| `+0x10` | ptr | Next entry in the same bucket |

**Map-pack clip dictionaries** (2026-11-23, confirmed on the minigame
pack): single-animation clips have the animation pointer at `+0x68` and
start / end / rate at `+0x90` / `+0x94` / `+0x98`. Short names are stored
inline when byte 15 has bit `0x80` set (length in the low 7 bits).

A full-archive walk of the bucket chains found 52,912 dictionaries and
294,794 entries. The bucket selection formula is unknown. The stem hash
is the lowercase joaat of the last path component without `.clip`. The
set hash is the lowercase joaat of the directory without `pack:/`
(12,626 records), or the dictionary name hash for flat
`pack:/<stem>.clip` paths (282,168 records). Both rules held for every
entry.

### Scene clip references

In the `0DRP` v122 class registry, each entry's `u32` at `+0x18` is the
**instance stride**, not descriptor byte length. Read the `u16`
descriptor count at `+0x1e`, then that many 16-byte descriptors from the
`+0x10` pointer. A packed object reference stores a one-based block index
in the low 20 bits and a byte offset within that block above them.

All 22,823 `0DRP` schemas in `anim.rpf` contain inline clip-reference
class `0x24749343`, whose instance stride is `0x18`:

| Offset | Type | Meaning |
|---|---|---|
| `+0x08` | `u32` | Dictionary/set hash; descriptor field hash `0x25de16df` (`clipDictionaryName`), type `0x4a` |
| `+0x0c` | `u32` | Clip stem hash; descriptor field hash `0x24fc8f24` (`clipName`), type `0x4a` |
| `+0x10` | `u32` | Flag; observed 695,621 times as 1 and 10,644 times as 0 |
| `+0x14` | `u32` | Observed zero |

To find the reference, do not assume a fixed offset in the event object:
search parent descriptors for a type-`0x05` field whose aux class tag is
`0x24749343`. Eleven parent class tags occur in the archive. Most use the
`clip` field (`jhash("clip")` = `0xe71bd32a`) at `+0x38`; four also have
a second reference (`0x7c0a2099`) at `+0x68`. Less common layouts place
it at `+0x08` or `+0x120`, or one inline level deeper. In the common
shared-header event classes, field `rate` (`0x7e68c088`) is an `f32` at
`+0x5c`.

A full-archive pass walked 1,628,388 reference instances, of which
706,265 had both hashes nonzero. The same pass carved 52,912 dictionaries
and 294,794 bucket nodes. Requiring a bucket node with
`(stem,set) == (clip_hash,dict_hash)`, plus the mirrored keys in the clip
record, fully validated 148,314 reference rows into 14,575 unique
dictionary/clip pairs, covering 6,856 dictionaries and 6,201 stems.

The unresolved rows are not simply corrupt: 482,552 name a carved
dictionary but not the stored clip stem, while 75,399 name a dictionary
hash absent from `anim.rpf`; 554,783 stems occur only under another set.
The dominant misses reference gesture variant sets whose stems are stored
under sibling/base/common sets, consistent with a fallback hierarchy.
Treat an absent direct `(dict,clip)` match as unresolved rather than
proof that no clip exists.

For example, the scene at `anim.rpf@0x94f010` contains pair
`0x65d46c10`/`0x9302c678`, selecting dictionary `anim.rpf@0x272d5010` and
clip `+0xdba60` (`pack:/jason_buddy_kiss_jason.clip`). That clip's source
window uses source `+0xdc830`, sequence `+0xa7390`, from 4.5 to 7.9
source seconds at rate 1. Dictionary `0x26eeb010` contains the same clip
name and stem hash under a different dictionary hash (`0x7cb407d0`), so
a name or stem match alone selects the wrong copy.

### Clip windows and source tracks

For the sampled type-2 clips, `+0x18` is a 16-byte name field. If bit 7
of byte `+0x27` is set, the NUL-terminated name is stored inline at
`+0x18`, with length `byte & 0x7f`. Otherwise `+0x18` points to the name;
the `u16` at `+0x20` is its length and the `u16` at `+0x22` is
`length + 1`. Across the archive, all 909 names of 12-14 characters were
inline, and all 293,885 names of 15 or more characters used the pointer
form. `+0x68` is the `f32` clip duration, and `+0x58` points to an array
of 0x18-byte wrappers counted by the `u16` at `+0x60`. Each wrapper has a
window array pointer at `+0x00`, `u16` window count at `+0x08` and `f32`
duration at `+0x10`. Window stride is 0x38:

| Offset | Type | Meaning |
|---|---|---|
| `+0x00` | ptr | Source animation record |
| `+0x08` | `u32` | Lowercase joaat of the source name without `pack:/` or `.anim` |
| `+0x28` | `f32` | Start time in the source, seconds |
| `+0x2c` | `f32` | End time in the source, seconds |
| `+0x30` | `f32` | Source seconds per clip second |

Sample each window at `start + clip_time * rate`. Different windows can
contribute tracks over the same clip time; do not concatenate them. In
836 clips across seven dictionaries, all 2,502 source pointers and hashes
resolved, and `(end - start) / rate` agreed with clip duration within
0.000013 s. An embedded `pack:/...clip` string alone is not proof of a clip
record.

For the sampled source record layout with a 0x20-byte prefix, offsets
from the **record start** are:

| Offset | Type | Meaning |
|---|---|---|
| `+0x10` | `u32` | Number of clip windows in the dictionary that reference the source, plus one |
| `+0x24` | `u16` | Source frame count |
| `+0x26` | `u16` | Frame stride between sequence starts in multi-sequence sources |
| `+0x28` | `f32` | Source duration, seconds |
| `+0x30` | ptr | Source name |
| `+0x50` / `+0x58` | ptr / `u16` | Sequence pointer array / count |
| `+0x60` / `+0x68` | ptr / `u16` | Track row array / count |

The `+0x10` relation held for all 2,411 sources in 618 sampled
dictionaries; 26 of those sources were used by more than 63 windows.

The track stride is **12 bytes**: `{u32 target_hash, u32 property_hash,
u32 kind}` (233 sampled tables). For skeletal tracks, the target is the
full lowercase joaat of the bone name, not the 16-bit tag. It matches the
`u32` at `+0x1c` in the [ped bone record](07-models.md#75-skeleton-fragments-confirmed).
Verified on the `ambient_malebody` and `ambient_femalebody` fragments at
`packs.rpf@0x454db010` and `packs.rpf@0x45404010` respectively.
Property `0x98e1750c` (`bonerotation`), kind `0`, is an XYZW local rotation;
`0xf791a95f`, kind `2`, is an XYZ local translation. Other track
properties can target controls rather than indexed bones.

The row kind gives the value shape of the track's channel group, not its
channel type:

| Row kind | Channel group |
|---|---|
| 0 | Quaternion: one type-0 static quaternion; three scalar components plus one type-7 marker; or scalar components 0-3 plus one type-8 marker |
| 2 | Vector3: one type-1 static vector, or scalar components 0-2 |
| 3 | Two scalar components, 0-1 |
| 4 | One scalar component, 0 |

All 94,092 rows decoded from 981 sequences matched this table. Only
these four kinds occurred in 12,620,439 rows from 190,318 sources. The
kind belongs to the row, not the property: `0x2deefadd` occurs with
kinds 4, 2 and 0.

Four kind-4 properties target ped bones. In the full table census,
`0x135d54f6` targets exactly Pelvis, Spine3, Head, both hands and both
feet in all 12,120 sources that use it; `0x56352445` targets exactly both
feet in 15,321 sources; `0xb3c29ff4` and `0xabd2feb8` target subsets of
the same seven bones. In 4,831 varying curves decoded from a
representative sample, every segment between resets started at 0 and
ended at 1 within 1e-6, and every reset fell within two source frames of
a clip-window boundary. Their runtime meaning is unknown. `0xb3c29ff4`
and `0x56352445` can store byte-identical curves for the same foot
(`anim.rpf@0xf7f1f010`, sequence `+0x20000`) or different ones: in step-turn
`anim.rpf@0x67b60010`, source `+0xdda0`, sequence `+0x55b0`, the left
foot reads 0 and 0.602151 at frame 334.

Sequence `i` starts at source frame `i * frame_stride` (zero-based).
Each non-final sequence stores `frame_stride + 1` samples: its last
sample and the next sequence's first sample share a source time. All
14,133 multi-sequence sources with validated headers satisfied
`sum(sequence_frame_counts) - (sequence_count - 1) = source_frame_count`.
All 228 non-final sequence headers checked against the stride field
matched. Across 44 decoded joins, scalar and vector samples agreed
closely; the type-8 handling below makes all sampled shared-frame
quaternion tracks agree.

### Sequence channels

The 0x20-byte sequence header gives the payload size at `+0x04`, the
payload-relative frame-data offset at `+0x0C`, and the header-relative
root-motion offset at `+0x10` (all u32). The frame count and bytes per
frame are u16 values at `+0x16` and `+0x18`. In all 307 cached
sequences, `+0x14` was 2 and `+0x1E` was 0x0b; their meanings are open.
The `u16` at `+0x1A` is the sum of `(table_words + 5)` over the type-5
channels, and `+0x1C` is the sum of the type-4 frame bit widths. The
frame stride is the total frame-coded bits (32 for each type-3 channel
plus the stored widths for types 4 and 5), rounded up to a 32-bit word;
this layout check passed on all 307 cached sequences.

Channel parameters precede the frame data. After
`frame_count * bytes_per_frame` bytes of frame data, read **eleven**
u16 channel-list counts and their u16 lists. A list word is
`track_index << 2 | component_index`; the track index addresses the
12-byte rows. In the sampled type-7 cached quaternion records, the low
two bits identify the omitted XYZW component, reconstructed as the
positive square root of `1 - sum(other_components^2)`. Type 8 is
different: the scalar channels store all four XYZW components, and the
type-8 marker consumes no parameter or frame bits. Read all four stored
values by slot index and normalize them for playback. Across 1,885
type-8 tracks (1,286,011 samples), the largest pre-normalization norm
error was 0.00137. Keeping all four values brought all 855 shared-frame
comparisons below a sign-invariant quaternion distance of 0.01, while
discarding one component produced 505 mismatches above 0.5. Pad each list
to a multiple of four u16 words. Header `+0x10` points to the first
six-byte root-motion reference: `+0x10 - 0x20` equals the end of the
channel lists in 563 footer sequences and the payload length in 448
sequences without footers. The high and low nibbles of byte `+0x1F`
count position and rotation references.

Each reference is `{u8 channel_kind, u8 component, u16
parameter_word_offset, u16 frame_bit_offset}`. The parameter offset is in
4-byte words from the start of the sequence payload; the frame offset is
in bits from the start of a packed frame. A group of references selects
the complete channel list of one target-0 trajectory track. In 1,126
groups, the position rows were `{target=0, property=0x7e58c0a0, kind=2}`
and the rotation rows `{target=0, property=0x103ca38a, kind=0}`; the
latter was always track-table index 0. Intersecting the candidate tracks
across each group resolved a singleton 1,126/1,126 times, and the
reference multiset matched the track's channel multiset.

Of 3,096 references, 329 were individually ambiguous. All were kind-7
or kind-8 markers at the end of a rotation group: markers consume no
parameter or frame bits, so same-kind markers can share offsets. The
group's other references identify the track.

**Type 5: indirect quantized float.** The parameter record is
`{i32 index_bits, i32 value_bits, i32 table_words, f32 quantum, f32 bias}`
followed by `table_words` `u32` words, so it occupies
`20 + 4 * table_words` bytes. Each packed frame stores an
`index_bits`-wide LSB-first index. The index selects an LSB-first
`value_bits`-wide entry from the bit-packed table, and the value is
`entry * quantum + bias`. In 668 type-5 channels from 218 sequences, the
header `+0x1A` total matched every sequence, every channel used a dense
index range starting at 0, and all bits after the table entries were
zero. Index `(1 << index_bits) - 1` never occurred, including in 229
channels whose table could hold it. All 118 shared-frame comparisons of
tracks containing type-5 channels agreed within 0.001.

**Type 10: incremental float.** The parameter record begins with a
`u32` size in 4-byte words (including the header), four descriptor bytes,
then `f32 quantum` and `f32 bias`. Descriptor bytes 0 and 1 give the bit
widths of the offsets and initial values respectively. Byte 2 is the
rounded residual payload length per lane in 32-bit words
(`lane_bits / 32`); byte 3 was zero in all 35,964 records across the
299 cached sequences used for that check.

Read the payload LSB-first. Divide the sequence into 64-frame chunks.
Read one lane bit offset per chunk, then one initial quantized value per
chunk. Read a unary width (zero bits terminated by one) for each group of
up to eight residual frames, excluding each chunk's first frame. Align
to 32 bits. The residual payload is eight lanes interleaved by 32-bit
word: lane `l`'s logical word `w` is payload word `8*w + l` after
alignment. Each residual group reads the same width at the same bit
offset from lanes 0 through `min(8, remaining_frames) - 1`, in order.

```text
for each chunk:
    pos = lane_offset[chunk]
    value = initial_value[chunk]
    velocity = 0
    emit(value * quantum + bias)
    for width, count in the chunk's residual groups:
        for lane in 0 .. count-1:
            u = read_unsigned_bits(lane, pos, width)
            velocity += (u >> 1) XOR -(u & 1)
            value += velocity
            emit(value * quantum + bias)
        pos += width
```

Each chunk's widths sum to the distance to the next stored lane offset.
The last lane end, rounded to 32 bits and multiplied by eight, accounts
for the rest of the record. Both checks passed for **36,158 records in
302 sequences**. Type 10 contributes no bits to the per-frame stream.
This layout produces continuous rotations in tested clips.

Large steps inside a source are not necessarily decode errors. Of 303
consecutive-frame quaternion steps above 0.5 on tracks containing type-5
channels, 297 were on the target-0 trajectory rotation track. In each,
the heading jumped to identity exactly on a frame where a catalogued
clip window starts or ends. Sample the clip window rather than the whole
source. The other six steps were on skeletal rotation tracks, and their
cause is open, as are other within-sequence discontinuities.

Types 6 and 9 remain undecoded, but neither occurred in the 212,302
unique sequences reached from 294,810 validated clips in 52,923
dictionaries. Type 3 occurred in only 30 of those sequences.

### Convertible roof clips

(2026-11-23, high.) `levels.rpf` holds 170 clip dictionaries as entries with
no recovered name, in the same layout as `anim.rpf`'s (every unnamed
object's system page was tested; `packs.rpf`, `data.rpf` and `common.rpf`
hold none). Eleven are `va_<model>` vehicle dictionaries (carbonizzare,
comet7, dominator9, dominatorclassic, jubilee, lasso, schutzen,
sentinelclassic2, vigero3, windsor2, zion2); the rest are map object clips
(names ending `_clip`).

Each convertible's dictionary has one clip, `roof`, with one animation
3.8–11.1 s long: 12–74 bone channels (rotation and translation), one mover
track and one float track. Every bone id is the joaat of a lower-case bone
name on the car's own skeleton (all ten map fully). Time 0 is the roof up,
the end the roof stowed; it drives the bones under `convertible_roof`,
including `roof_cover`. `va_jubilee` instead holds one clip,
`sunroof_r_anim` (1.67 s, 2 bones). The model info names the dictionary and
clip ([§11](11-vehicle-paint.md#vehicle-tables)). Open: what the float
track carries; whether hardtops or other vehicles keep clips under other
dictionary names.

### Pose root and trajectory

In `pack:/1f_twerk_ped_01.clip` (`anim.rpf@0xf7f1f010`, clip `+0x194f0`,
sequence `+0x20000`), `SKEL_ROOT` has a rotation and translation
separate from the trajectory in the sequence footer. Applying the two
skeletal tracks as **absolute local** transforms on the matching female
rig, rather than re-anchoring the first sample to the root bind
orientation, reproduces the planted portion of the motion without foot
IK. Over frames 39-100, the left/right toe-height standard deviations
dropped from 0.190/0.207 m with the re-anchored root to 0.005/0.001 m
with the absolute root, keeping all non-root keys unchanged.

In this example, the three position references resolve to track 36
(`0x7e58c0a0`); all four rotation references resolve to track 0
(`0x103ca38a`). Track 153 (`0xbd8d191d`) duplicates the heading curve but
is not referenced by the footer. The trajectory reaches 0.728 m of
planar displacement and about 141 degrees of heading change. How the
runtime combines it with the pose is not yet verified.

**Mover tracks** (2026-11-24; names and values high, names re-hashed with
joaat): tracks with bone id 0 are the **mover**. Kind `7e58c0a0` is
`moverabsolutetranslation` (vector3, metres, Z-up, forward +Y) and kind
`103ca38a` is `moverabsoluterotation` (quaternion, a turn about Z). Both are
absolute over the whole animation, so a clip cut from the middle of an
animation starts away from the origin; the movement since the clip's start
is inverse(start) × current. The bind pose of a character faces −Y, and
a walk clip's bone tracks turn it to face +Y.

- Check (one dictionary): on a female walk clip (3.70 s, 6.65 m, no turn),
  adding the mover to forward kinematics keeps each planted foot within
  1 cm of the ground at four contacts per foot; without it the foot slides
  about 0.7 m per step. Two turn clips of the same dictionary turn 346°
  left and right. *This answers, for these clips, how the trajectory
  combines with the pose: the mover is applied on top of the bone tracks.*
- Open: five more mover track kinds in the same animations, not named or
  decoded: `60d0644c` (float, zero in the sample), `765c7d5f` (vector3),
  and the quaternions `a8219307`, `af65ca97` and `bd8d191d` (nearly equal
  to the absolute rotation, as track 153 above).

## Ped body shape

Body shape is a **sparse additive pose rig** on helper bones (`BM_*`,
`MH_*`, `SEC_*`, `RB_*`) of the shared ped skeleton
([§7.5](07-models.md#ped-skinning-bone-palette-confirmed)); the clothing is
skinned to those bones. Three pieces:

**Presets** (plain XML in `common.rpf`, `CPedVariationBodyTypeManager`):
height presets (1.52–2.02 m), scale profiles and 16 body-shape presets
(THIN, HALFTHIN, REGULAR, HALFHEAVY, HEAVY, MUSCLE*, BELLY*, HEAVY_EXTREME,
DANCERFEMALE, LINEBACKERMALE …). Each preset is a list of controls:
`gcweight<region>` (−1 thin to 1 heavy) and `gcmuscle<region>` (0 to 1) for
back, belly, butt, calf, chest, forearm, hip, muffin, neck, shoulder,
thigh and upperarm, plus `GCBreasts`, `GCCleavage` and `GCButt`. REGULAR is
all 0 (confirmed).

**Controls → rig inputs** (a preset PSO in `anim.rpf`: 922 of its 8,192
items carry rig inputs; confirmed). A negative `gcweight` drives the region's
thin input by |v|, a positive one its heavy input by v; `gcmuscle` drives one
input directly.

| Region | Heavy | Thin | Muscle |
|---|---|---|---|
| back | `722b3991` | `e79422cf` | `083e8443` |
| belly | `efc858c0` | `74a5940e` | `860a629a` |
| butt | `b65f7951` | `1d70bfc2` | `2267feeb` |
| calf | `4c92b04a` | `b3925c52` | `798aa11f` |
| chest | `fbc18785` | `47164eb7` | `f8ca1901` |
| forearm | `7e5f130b` | `49c5ac98` | `33c4e1ea` |
| hip | `0978bf99` | `596b89f1` | `fc1aaed3` |
| muffin | `b0c81e7f` | `9ffec641` | `3f088cd7` |
| neck | `334633c1` | `efc4dbff` | `97d111cf` |
| shoulder | `6d8264ca` | `fb039566` | `ab82da7b` |
| thigh | `525a6395` | `7636ca63` | `9e37dc2e` |
| upperarm | `c8a41369` | `5a7029e5` | `37657a6b` |

`GCButt` drives `5a502ae2`; `GCBreasts` drives `377b0101` when positive and
`48f90678` when negative. The input names themselves are unresolved.

**Rig dictionaries** in `anim.rpf` (magic `RIG` v2 at +0x70): one with 1,256
rigs (`anim/rig/generated/…`) and one with 543 (`anim/rig/canonical/…`).
Header +0x48 points to 0x50B hash buckets; each entry is {u32 joaat of the
full rig path, u64 pointer to a 0x30-byte wrapper (+0x10 count 1, +0x20
pointer to the rig), u64 next}. Body rigs include
`LOD0`–`LOD3_Body_Shapes_Female`, `Body_PoseDeformation_Female`,
`Dancer_Body_Shapes_Female`, `BreathingStyle0_Body_Shapes_Female`, XXL and
bodybuilder variants and garment rigs (`*_Garment_Shapes_Female`).

A rig is wrapped as `RIG` + byte 0x02, a u32 size (= the inner blob's
size), and the `RIG0` blob at +0x100: version bytes, size, a section count
at +0x10 and section offsets from +0x14. Sections are tagged {u32 type hash,
u32 size}:

| Section | Contents | Confidence |
|---|---|---|
| `e636c3e9` | block table: count, register count (1,252 floats on `LOD0_Body_Shapes_Female`), then u32 entries `type << 24 \| offset` into the program body; type 0xFF ends | confirmed |
| `2316833f` | inputs: records {u8 type, u8 kind, u16 register, u32 hash}. Kind 3 = a bone track (joaat bone name; type 0/1/2 = translation / rotation / scale), kind 1 = a float input | confirmed |
| `7224f987` | outputs: the same bones | confirmed |
| `b00f0959` | the program (about 25 KB); its body starts after type, size and u32 `0x00020000` | confirmed |
| `74d00508` | 5 named blocks {hash, offset, size} | inferred |
| `6257c371` | the rig path | confirmed |

Program blocks:

| Type | Block | Confidence |
|---|---|---|
| 0x85 | constants: u16 first register, u16 count, then f32 values (registers 0, 1 = 0.0, 1.0) | confirmed |
| 0x46 | u16 dst, a, b: probably dst = a × b | inferred |
| 0x81 | u16 dst, src, then a 0..1 range: probably a clamp | inferred |
| 0x86 | one pass: u16 counts nT, nS, nQ, 0; then nT + nS + nQ u16 triples (track register, weight register, the track register again); then the deltas, 3 floats per translation and scale entry and 4 per rotation entry (unit quaternions). Each entry does T += w·dT, S += w·dS or q = q·dq^w on the track, in place; a track appears at most once per pass | confirmed |

`LOD0_Body_Shapes_Female` has 12 passes (360 down to 3 entries) over 360
helper-bone tracks (translation vec3 at registers 8–429, scale vec3 at
430–775, rotation quaternions at 776–1229) and 53 float inputs. Most deltas
are on scale. The tracks are also inputs, so the rig adds to the animated
pose. `LOD0_Body_Shapes_Male` has 1,294 registers, 379 tracks and 53
inputs, 50 of them shared with the female rig, all the control inputs
among them.

**Axes.** Deltas are in the game's Z-up bone frames. For a glTF skeleton
converted with (x, y, z) → (x, z, −y), convert translation deltas to
(x, z, −y), scale to (x, z, y) and quaternions to (x, z, −y, w). With that,
all 196 left/right translation pairs mirror exactly, and the shapes look
right: heavy widens torso and pelvis and pushes the belly forward, thin
slims (checked by eye on a female top).

Related: about 210 resources in `packs.rpf` with hashed `GM_U01_*` bone
references are soft-body / secondary-motion data, not body shape: particles
(x, y, z, w with w 400–3000) and edges {u16 a, u16 b, f32 rest length, f32,
u32 flags} (inferred). They are the soft-body variant of the cloth
resources in [§19.3](19-ped-physics-resources.md#193-cloth-type-2) (702 resources).

Open: the exact 0x46 / 0x81 ops, the lower-LOD rigs, the input names, and
why the belly spikes on some male T-shirts whose palettes name bones in
neither skeleton.

**GPU-side leads** (strings and counts measured; the reading is a guess):
compiled-shader strings name `ped_morphable_model_cs` (`CS_MorphableInit`,
`CS_MorphableShapeSolve`, `CS_MorphableReconstructNormals`, next to
`ped_skinning_cs`) and `ped_standard_head_cs`
(`CS_ComputeBlendshapeMappings`, …), and skinning vertex types include
`pedblendshapes`, `pedsecmotion` and `deformable*`, so body shape is
probably also solved per vertex on the GPU. Ped mesh blobs carry slots 17
(format 0x84, 16 bytes) and 19 (format 0x27, tangent-like), maybe packed
morph data. The preset PSO in `anim.rpf` (`@106153000`, 1.9 MB) is a hashed
dictionary of 15,793 scalars over 322 names (all 27 body controls plus 295
unknown, probably face), 932 vec3 and 466 vec4 values. Idle and locomotion
clip dictionaries key the body controls (`gcmusclechest`, `gcweightbelly`)
as animated tracks. The 53 rig input names match none of these names under
joaat, case-sensitive joaat, FNV-1a or common prefixes.

## Music catalogue

`music.dat` is compiled parsable data: a root count, then the magic u32
`0x045A88EA` at `0x08`, then type-tagged records with `0xAAAAAAAA` as the null
sentinel (confirmed).

A title record (tag `0xAAAAAA0C`) has a u32 title hash at `+0x08`, a char[64]
title at `+0x0C`, a u32 credit count at `+0x60`, then 8-byte credits
(confirmed).

The title hash is joaat of the **lowercased** title with seed `0xC3CCF6AD`
(804/804 titles, confirmed). AWC stream ids use unseeded joaat of the title, so
the two never coincide.

## Arcade values in `common.rpf` XML

(2026-11-23; counts and structure high.) `common.rpf` holds 208 plain XML
files; find them by root element, not by position:

| Root | Arcade content |
|---|---|
| `timecycle_modifier_data` (two files) | 43 modifiers with "arcade" in the name: 34 `ARCADE_*`, 8 `ArcadeGame_*` / `ArcadeShooter_*`, 1 `Overlay_ArcadeCabinetCRT`. Each has a name, `groupApplyTag` (`final` or `animpostfx` in the ones read) and child elements whose text is two numbers |
| `CPedInventoryLoadOutManager` | 8 `LO_AG_TSS_*` loadouts: a weapon preset name plus flags |
| `CDamageDataManager` | 33 `Arcade_*` damage rules (10 `Arcade_TwinStick_*`, 23 `Arcade_BeatEmUp_*`), typed by a `type` attribute (clamp, multi-reference, conditions) |

Open: the meaning of each timecycle value's second number, which modifier
applies to which game or moment; the weapon presets' own numbers are not in
these files. The full timecycle format is in
[Timecycle and weather](#timecycle-and-weather).

## Timecycle and weather

(2026-11-23; checked against an independent XML parse: 32,742 modifier
values and 59,170 cycle rows, 0 mismatches.) The timecycle data is plain XML
in `common.rpf`, so every variable has its name. The archive stores no file
names: identify documents by root element (`timecycle_modifier_data`,
`timecycle_keyframe_data`, `time_data`, `CContentsOfWeatherXmlFile`,
`CustomClouds`). Some entries hold several XML documents back to back (one
holds three area files) (high).

**Modifiers** (high): 4,593 in 50 files. Attributes: `name`,
`description`, `groupApplyTag`, `localApplyIndex`, `userFlags`, and
`numAxes` / `numKeyframes` when animated; a modifier may have a
`<submodifiers>` list. Each variable holds two numbers: the first is the
value; the second is 0 in 88 % of records and 1 in 10 % (also 5000, 100,
73 …), meaning unknown. Animated modifiers have `<axis controlValue
blendType>` blocks with one number per keyframe (times in
`keyframe_t_value`). Weather modules have `input_min` / `input_max` /
`output_min` / `output_max` blocks for high-level weather variables (11
blocks: fog, gradient, rain chance). One variable is written twice with
different numbers (`water_clip_extinction_b`: 150, then 200); which copy
the game uses is unknown.

**File names:** `TC_Files.meta` lists 50 `TIMECYCLEMOD_FILE` names plus
`UNDERWATER_DEEP.xml`, which is a keyframe file. Files are packed in
case-insensitive name order; aligning that order with each file's modifier
names (an area code prefix or a word of the file name) maps all 50 (high for
the area and arcade files; by order only for `timecycle_mods_1`–`4`).

**Weather cycles** (high): 27 keyframe files (`W_<NAME>.XML` from the
weather list; `sleet` is not in that list). Each has `GLOBAL` and `URBAN`
regions and about 1,100 variables, with 13 samples per variable at hours 0,
4, 5, 6, 7, 9, 12, 16, 18, 19, 20, 21 and 22, each with an interpolation
type and `uw_tc_mod`. There are 1,426 distinct variable names; families
include postfx 376, local 240, light 140, sky 104 and fog 15.

**Arcade:** `TC_mods_Arcade_Games.xml` has the 34 `ARCADE_*` modifiers.
`Overlay_ArcadeCabinetCRT` is in `TC_mods_APFX.xml` (`groupApplyTag`
`animpostfx`): `postfx_noise` 1.0, `postfx_noise_size` 0.6,
`postfx_exposure_perpixel_amount` 0.75. The `CustomClouds` group
`arcade_TwinStickShooter` removes the clouds and adds its own. The minigame
pack has no timecycle data.

Open: the run-time strength and blending of modifiers, the second number
and `userFlags`, and what the engine does with each variable.

## Script data documents

(2026-11-23; structure high.) `script.rpf` holds 1,012 PSO files that are
**binary XML documents**. An element has a name hash, first child, next and
previous sibling, parent, first attribute and attribute count. An attribute
has a name, a type (1 vector index, 2 f32 bits, 3 i32, 4 bool, 5 string
index) and the joaat of the value text. With the current dictionary 36 % of
names and 7 % of string values resolve. The string block and the compiled
script code (51.7 MB in `story000_script.rpf`) have an entropy of
7.85–8.0 and no compression headers, so they look encrypted; not decoded.

**Resource lookup tables** (purpose unknown): two Swiss-table hash maps, 16
control bytes equal to each key's low byte, then 32-byte slots holding a
u64 key and a value pointer.

## PSIN scene placements

64-byte records at a 16-byte stride inside PSIN tag blocks: 97,504 records in
2,017 documents of `story000_script.rpf` (confirmed).

| Off | Meaning | Confidence |
|---|---|---|
| 0x00 | u32 handle, repeated at `0x30` | confirmed |
| 0x10 | f32 × 3 position, Z-up metres; x repeated at `0x1C` | confirmed |
| 0x20 | f32 × 3 rotation, radians; x repeated at `0x2C` | inferred (meaning) |
| 0x34 | u32 archetype = joaat(lowercase name) | confirmed |
| 0x38 | u64 id; zero in about half the records | inferred (meaning) |

These are a different family from the map-metadata entities in [§9](09-placements.md).

## World height map (`MHIU`)

(2026-11-24; layout high, decoded and checked against the data.) A
whole-world surface height map in `common.rpf`. All fields are little
endian.

| Off | Meaning |
|---|---|
| 0x00 | magic `MHIU` |
| 0x04 | u16 version (1) |
| 0x06 | u16 width (2048) |
| 0x08 | u32 row count (2048) |
| 0x0C | f32 × 6 world box: min x, y, z, then max x, y, z (−13000, −9000, −50 / 7000, 11000, 500) |
| 0x24 | u32 byte count after the header (row table plus samples) |
| 0x28 | u8 fill value (`0x17`, sea) |
| 0x29 | 3 bytes, unknown; they differ between the two copies (the third may be a checksum, a guess) |

- **Row table** after the 0x2C-byte header, 8 bytes per row: u16 first
  column, u16 count, i32 offset. The offset is biased by −first, so column
  x of the row is `samples[offset + x]`. The counts add up to the sample
  bytes exactly.
- **Samples** are u8. Row 0 is the south edge and column 0 the west edge;
  each sample covers 9.77 m. Height = min_z + v × (max_z − min_z) / 255, so
  one step is 2.16 m and the fill value `0x17` is −0.39 m. Cells outside
  every run are sea.
- **Two copies** on the same grid (about 1.2 MB each) store 1.17 M and
  1.19 M samples. Of the samples both store, 892 K are equal and 139 K
  differ by +1 step; some land exists in only one copy. What sets the two
  apart in game is unknown.
- **Against the collision top surface** ([§13](13-collision.md), 8 m grid,
  about 750 K land samples): median difference about −2 m; the second copy
  has 72 % within 3 m, 96 % within 10 m and correlation 0.993, the first
  copy correlation 0.974. It is a surface map: buildings show up.

## Text (GXT)

GXT resources are `0DRP` type `0x7a`. A tag table points at a `0x101` key
table, `0xA8478889` locators, `0x956A9DF5` / `0x10075946` locale-variant groups
and items, and `0x10` text pages (confirmed). Keys are joaat, stored in 4,096
or 2,048 buckets (confirmed).

### Key lookup

Read the 16-byte descriptors backwards from the root minus 0x10 until
the locator descriptor is reached, then reverse them to get the
text-page order. Each descriptor is `{u32 tag, u32 size, u64 pointer}`.
Pointers use the system base `0x50000000`.

1. Hash the ASCII-lowercased identifier with unseeded joaat. A
   0x110-byte bucket has 16 fingerprint bytes followed by 16 records
   of `{u32 key, u32 zero, u64 value}`.
2. Start at `(key >> 7) & (bucket_count - 1)`. Compare the full key
   in slots whose fingerprint equals `key & 0x7F`. If none matches,
   stop on a bucket containing an empty slot (`0x80`). Otherwise advance
   one bucket, wrapping at the end. Bound the probe by the bucket count.
3. In the matching row, `value & 0xFFFFFF` is the kind and
   `value >> 24` is a slot in units of 16 bytes. Kind 1 points into
   the direct locator table. Read its u64 text locator at row `+0x08`.
   Kind 7 points into the variant-group table.
4. Decode a text locator with `page_index = (locator & 0xFFFFF) - 4`
   and `byte_offset = locator >> 20`. Resolve the resulting offset
   inside its text page to a NUL-terminated UTF-8 string. The offset
   must be the start of the page or follow a NUL, and the terminator
   must be inside the page.

A variant group is 0x20 bytes. It holds u32 count and capacity at
`+0x08` and `+0x0C`, and a u64 item reference at `+0x10`. The
reference's low 24 bits are 8. Its high bits give the first slot in the
variant-item table. Item `i` is at `16 * (first_slot + 2 * i)`
bytes into that table, with its text locator at `+0x00` and u64
attributes at `+0x08`. This lookup recovers the stored alternatives.
The runtime selection rule for the attributes is unknown.

### Tables, labels and markup

(Confirmed by decoding every table, 2026-11-22.) The resources carry no
name. `data.rpf` holds **14,482 tables with 4.21 M strings** (UI, subtitles,
item names, phone, ambient speech); `pack/shared_minigame000` holds 78
tables with 23 K strings: 13 languages × 6 tables (arcade minigame
subtitles of 322, 417, 490 and 505 strings, 23 speaker names, a 12-line
poem). Each table is **one language**. The language is not stored; it
follows from the text. Copies of one table sit at fixed strides through
`data.rpf`.

- The block table at +0x38 ({u32 struct hash, u32 size, u64 pointer}, u16
  count at +0x4C) has the `a8478889` locators (16 bytes each), the `0x101`
  key map and the `0x10` text page, which sits before or after the map
  depending on the file. Read keys only from the `0x101` block: a scan of
  the whole file for slot-shaped records finds false hits (e.g. key
  `00000006`).
- **Big tables split the text over two or three `0x10` blocks** of up to
  1 MiB each. A string reference's u64 at +8 is `(offset << 20) | block`,
  with the block counted from 1 in the block table; reading only the first
  text block gives wrong strings for about a third of a big table.
- Labels are joaat of the lower-case label. A **subtitle's label is its
  voice-line name plus a 2-digit variation**, e.g. `AG_SUR_GEAR_FIRST1_01`.
  The minigames' dialogue PSO (`shared_minigame000.rpf@19000`) keys lines by
  joaat of the line name → {`8d6b17bd` (`BankName`), `MissionId`}. Most label names
  are not recovered.
- Inline markup: `~sl:gap:duration~` splits a subtitle into timed pieces
  (seconds); `~n~` new line; `~b~`, `~s~` colours; `~1~` parameters;
  `~INPUT_…~` and `~INPUTGROUP_…~` button icons, `~BLIP_…~`, `~COLOR_…~`
  (`COLOR_SUBTITLE_FOREIGN_DIALOGUE` marks translated foreign-language lines).
- **Minigame subtitles** (2026-11-23, high): the pack's conversation table
  entries read straight from the PSO bytes as `AG_<game>AU_<4 letters>\0
  <event name>_<NN>\0`; linking voice files to events through it gives
  1,159 subtitled voice files across the four arcade games (321, 299, 272
  and 267).
- **The puzzle game's own table** is `data.rpf@1965010` (181 strings): menu
  and pause entries, control hints, credits, track titles, and 11 notes,
  10 of them in 11 variants that differ by one emphasised keyword, so the
  game swaps a keyword per playthrough or state (structure high).
  `data.rpf@19621d0` has 5 strings (three arcade titles); `data.rpf@19be010`
  is the big general table (36,888 strings). The puzzle table was only
  reachable once multi-block tables were read correctly (above).

---

← [16. Road graph, junctions and flags](16-roads.md) · [Index](README.md) · [18. Radio station audio and scheduling](18-radio.md) →
