# 11. Vehicle paint and vehicle tables

*Part of the [SixNG format specification](README.md). Numbers as of 2026-11-21; confidence: confirmed / inferred / unknown.*

| Item | Finding | Confidence |
|---|---|---|
| colour slot `c8e8c282` (`diffusetex`) on vehicle shaders (`standard_vehicle_2lyr`, `4aa38282`) | points at an engine-default **white** texture found in no store; the body colour is a runtime paint colour, as in GTA V | confirmed |
| `f151c063` | decal / sticker layer, RGBA | inferred |
| `d37026cb`, `640111c6` | dirt / grime / wear masks | inferred |
| `84d22f74` | fine scratches and flakes | inferred |
| `b9f4bdac` | grime | inferred |
| `fb03ae36` (`dusttexture`) | green-dominant mask (mask pack or normal detail) | unknown |
| `4d56d7fe` / `2bb2ede8` (`lyr1materialatex` / `lyr0materialatex`) | BC1 atlas, normal-like. *Later (2026-11-23, high, one car): **not a normal map**.* R and G are exact complements (R + G = 1, mean error 0.005, correlation −0.998) and only 54 % of texels are unit length (100 % on the real normal map): one signed edge / cavity signal stored as R = x, G = 1 − x, about 0.5 on flat areas. The pixel shader uses R as the metal blend (`metalLow` → `metalHigh`) and G as the roughness blend (`roughnessLow` → `roughnessHigh`); B is an **occlusion factor** multiplying both the diffuse-side and the specular-side colour at the end of the pixel shader (high); A is read once and unused in the variant read (medium, one variant) | inferred; later high |
| `84c69350` / `58b6e707` (`lyr0normaltex` / `lyr1normaltex`) | BC5 normals | confirmed |
| shader constant buffer (+0x08) | ~0x160 bytes of floats; *keyed by the parameter table's constant descriptors (2026-11-24), see [material constants](07-models.md#material-constants)* | layout confirmed; values per field unknown |
| `19df25d7` | 128 × 256 R8G8B8A8 sRGB texture of horizontal colour bands; one texture (same hash) is bound by every two-layer material of a car and by the shared tyre model (12 of 12 `standard_vehicle_2lyr` materials on one SUV, plus the tyre). Probably a material palette indexed by the layer maps or constants; the lookup is unknown. *Later check (ambulance and SUV): every vehicle binds the **same** texture: row 0 R / G / B markers, rows 1–32 black, rows 33 and up ramps, so it is probably a placeholder the game replaces (inferred); this disagrees with "one per car" and is kept alongside it.* **Later still (2026-11-23, high): not a placeholder but the shared material palette**; row = a colour record's palette row, see [part colours](#vehicle-part-colours). **Array of 8 slices (2026-11-23, high):** the texture is 128 × 256 × 8 slices; the descriptions above looked only at slice 0. A colour record's Page is the slice, Selection the row ([slices](#vehicle-library-field-names)) | format, sharing, array confirmed; role high |
| `346394ea` / `7d4f12e2` (`lyr0diffusetex` / `lyr1diffusetex`) | **linear BC7 detail maps, not colour**: R and G grey around 0.5, B a sparse mask. *Base colour slot (2026-11-23,
high that it is data):* on two-layer vehicle materials the base colour
slot often binds the **same texture** as `lyr0diffusetex` / `lyr1diffusetex`
(linear, mean about 128, 120, 21, alpha unused at 250 or more); it is not an
albedo, and shown as one it gives olive parts. Whether an sRGB texture in
that slot is a real colour is assumed from its format only. *Later (2026-11-23, high):* R 0.50 ± 0.04, G 0.47 ± 0.05, B mostly 0; G (plus a detail term) is the **palette column**, so most texels sample near the middle of a palette row; the shader does not flip or remap the column for any palette page | confirmed (not colour; channels on one map) |
| COLOR0.a on `standard_vehicle_2lyr` | a **part id** (0–255, a few values per material), not a layer weight | confirmed (ambulance, SUV) |

### Vehicle shaders (2026-11-22)

| Shader | Role | Confidence |
|---|---|---|
| `4aa38282` `standard_vehicle_2lyr` | **not only body paint**: one SUV has 12 materials on it, the largest (82 K triangles) spanning the whole car including the interior. Rule of thumb (inferred): a material is body paint when it binds no detail texture and its meshes sit on body bones. Real per-part colours are not decoded | confirmed (coverage) |
| `99ad6fc9` `standard_vehicle_2lyr_tyre_inst` | tyres; `diffusetex` is the engine-default white | confirmed |
| `c5cdbdd2` `standard_vehicle_2lyr_wheel_inst`, `99ad6fc9` | rims and tyres embed one **129 × 1 tint palette**, `584f3a883128c1a4`, identical on every rim and tyre looked at: entries 0–7 pure red, yellow, green, blue, magenta, grey (85), orange, cyan; 8–63 shades in groups of four; 64–128 one pink. Probably a placeholder the game overwrites with the wheel colour (inferred from the marker colours and the sharing) | palette confirmed; role medium |
| `389206d1` `standard_vehicle_alpha` | lamp lenses and clear covers, alpha bucket. `diffusetex` RGB is one constant colour (128, 128, 98 on the sample), opacity in alpha (78–255); also `extinctionlut` and `refractionbumptexture` slots. 41 materials in a 20,000-model sample | confirmed on one vehicle |
| `03b27476` `standard_vehicle_2lyr_reflector` | lamp reflectors (earlier grouped with car paint) | inferred (name, position) |
| `29648098` `standard_vehicle_emissive_animated_bones`, `5a693bbe` `standard_vehicle_emissive_layer` | lamps and lit panels; `emissivetex` (`8e0d63ee`) and `diffusetex` bind the same texture | confirmed on one vehicle |
| `5a693bbe` **runtime screen overlay** | a dashboard screen is two meshes about 1 mm apart: behind, `standard_vehicle_emissive_layer` with a real colour map; in front, the same shader with `diffusetex` and `emissivetex` both `a800afccf01aa36b` (in no store), always in the cutout bucket (record flags `0003a80108ff0000`; `0003a00108ff0000` on generic models). 570 models bind it (284 `_hi`, 275 `_mid`, 11 others; also on `standard_emissive_layer`); of the 284 `_hi`, 251 in both slots, 31 have no emissive slot, 2 another emissive map, and 267 have a textured emissive mesh behind it. Probably the texture the game draws dials and navigation into at runtime, or a transparent default; what is drawn and its size are unknown | medium |
| `bd71bdbf` `windshield_wipermask` | **wiper sweep mask**: no textures, one UV set, bound only to the `windscreen` / `windscreen_r` bones; u takes only {0, 0.014, 0.5, 0.986, 1} and v runs 0–1, the sweep coordinates of a fan over the glass. 210 of 284 surveyed `_hi` vehicles have one. Not a visible surface | high |
| `fc34f08b` `vehicle_water_stencil` | no textures, opaque bucket (record flags `0004200010ff0000`, `0005200020ff0000`): a lid over a boat's hull opening (74 meshes in the surveyed boats and jet skis), probably stencil-only; how the game uses it is unknown | medium (name and shape) |
| `fa9cc069` `standard_vehicle_glass` | `diffusetex` is a 64 × 512 gradient, constant along u; v selects clear glass (alpha 128) or the dark border (alpha 255) | high (two vehicles) |

## Vehicle tables

The game-data meta resource in `packs.rpf` (type 0x7a, 26 MB) and about 330
per-vehicle pack resources with the same structs (rows repeated) hold the
vehicle tables. Field names are case-sensitive joaat and match GTA V's
`vehicles.meta` names where they exist (confirmed).

| Struct | Count (game-data resource) | Fields |
|---|---|---|
| `e59aa857` model info | 327 | `gameName` (`120e5836`, usually joaat of the model name, see below), `handlingId` (`ec41bd9c`, the model name), `txdName`, `wheelScale` (`933301be`), `wheelScaleRear` (`76706238`), `wheelType` (`a2e90c68`), `layout`, `HDTextureDist`, `lodDistances`, `vehicleClass`, `swankness`, `plateType`, `defaultBodyHealth` … (52 names resolve) |
| `51019114` configuration | 3,879 | `fb1f80ff` model (field name `handling`), **`5bdc13e3` body colour** (u32 colour; first read as ARGB, but red is probably in the low byte, see [byte order](#vehicle-part-colours)), `e89b8540` part models (joaat names, about 150: frame, bumpers, exhaust, `brakedisc_*`, `caliper_*`, and one `rim_*` and one `tyre_*` in 82 %) |
| `fc1c7000` variation set | 4,772 | `fb1f80ff` model (`handling`), **`8c866829` (`colours`) weighted paint-colour list**: 1 to 400 colours (same type as the body colour) with repeats (2,246 sets; this is the pool of 17,632 colours), `59a59c3e` (`slots`) weighted part groups: `37a6f9ef` slot, `a12ff3a2` (`options`) list of {`8e6612bf` part, `dc732774` (`probability`) weight %} |

With the per-vehicle packs this covers 373 models (305 named), 4,063
configurations (3,339 with a rim and a tyre) and 2,377 colour lists. Handling is
a separate PSO ([§17](17-other.md#vehicle-handling)).

- **Multi-vehicle tables** (2026-11-23): variation sets (`fc1c7000`, with
  the owner link to a vehicle's part set `f14244fd`) and configurations
  (`51019114`) are not only in single-vehicle packs. Both name their
  vehicle, and some vehicles have them only in tables that hold several
  vehicles; reading those too gives 320 vehicles with a part set instead
  of 319 (the new one has 12 configurations; high: no vehicle lost a
  configuration). A vehicle's configurations can also sit in more than one
  resource, so gather them per owner and apply modifiers once to the base
  set: doing so changed 2 vehicles (5 configurations) that had been
  modified twice (count high; that the game applies them once is medium).

- **Joining model info to a model** (2026-11-23, high): `gameName` is not
  always the model name: 46 of 325 distinct records hold a different value
  there than in `handlingId` (case-sensitive joaat). `handlingId` is what
  configurations and variation sets use as owner, and it matches the
  drawable name. Falling back to `handlingId` when `gameName` finds nothing
  raises the models with model info from 312 to 357 of 373; every surveyed
  vehicle with wheel mounts then has a record except two damage variants
  and one train. Open: whether several models can share one `handlingId`,
  and what `gameName` means where it differs (it looks like a display or
  make name); field `1d2a64a8` also equals the model name hash on some
  records.
- **Display names** (2026-11-23, high: checked on about ten vehicles, the
  make + name pairs are the expected ones): a vehicle's display name is in
  the [text tables](17-other.md#text-gxt) of `data.rpf` under the label
  equal to `gameName` (joaat of the lower-case model name); its make is
  under the value of `vehicleMakeName` (`3db8df3e`, case-sensitive joaat).
  No separate label string is stored. 321 of 373 models have a name
  string and 312 a make string (those without are mostly trailers and farm
  implements). A label occurs in 10–13 tables, one per language; the first
  in file order that has it is English. Model generations are told apart in
  the name text itself (a year prefix in English, a year suffix in at least
  one other language). Open: for the 46 records whose `gameName` differs
  from `handlingId`, which text the game shows.
- **Mod parts** (2026-11-23, high: parts placed at the named bone's bind
  pose sit correctly on two cars, and kit names resolve for all 12
  two-element kits of one car) sit in the vehicle's own pack resource (the
  0x7a resource with one model info):
  - `0caef606` is a part or a kit. `e6e17f14` (`name`) is joaat of the part
    model's name or of a kit name; a kit name is its left and right models'
    names without the side word (`comet7_exhaust_histr_19` for `…_lr_19` and
    `…_rr_19`; side words `_l_`, `_r_`, `_lr_`, `_rr_`). `37a6f9ef` is the
    slot: joaat of `bonnet`, `bumper_f`, `bumper_r`, `exhaust`, `spoiler`,
    `splitter`, `diffuser`, `towing`, `chassis`, `roof` (`bc96a7e5`) or
    `livery` (`805d8f10`). `7e5d1612` (`elements`) names `77f2b89e`
    records; `61e1e572` (`features`) is a list of hashes.
  - `77f2b89e` is one model of a part: the low u32 of u64 `079baba7` (`attachBone`) is
    joaat of the bone the model is authored around (`chassis` `d418625e`, the
    car's own space; `boot`, `bumper_f`, `mod_spoiler_attach`,
    `mod_spoiler_attach01`, `mod_attach_exhaust_lr` / `_rr`); `2b490ab7`
    lists the stock bones it replaces (a bumper lists `bumper_f` and its
    indicator bones); `8605bf0e` is a 64-bit value per model (on one mod
    element per vehicle it is the name of the vehicle's part set, see
    [part colours](#vehicle-part-colours)).
  - One sports car has 123 records, 31 with elements, 43 elements for its 43
    part models; 323 of 373 vehicles have parts (7,915 records with
    elements). Open: slots `46268fee`, `9bbfb64c`, `ac89920d`, `5a4394ab`
    and others.
- **Shared part models and truck bodies** (2026-11-23; high for the library
  chain: 16 trucks and pickups render with the right body; 6,084 shared
  parts with models over 316 vehicle records):
  - A configuration's part list names `0caef606` part records both in the
    vehicle's own pack **and in the shared vehicle library**. In the
    library a `77f2b89e` element is 48 bytes and its bone `079baba7` is a
    plain u32 (the low half of a u64 in the packs).
  - In the library the **low 32 bits of `8605bf0e` are joaat of the
    element's model name**: 1,996 of 2,027 elements name an existing model
    this way. In a vehicle's own pack the low half often matches no model
    name (see the part-set reading above). *Answered (2026-11-23, high):*
    in the packs `8605bf0e` is a **64-bit asset id**: the first half (h1)
    of MurmurHash3 x64 128, seed 0, over the model's lower-case name
    without LOD suffix. All 69 elements of one pickup's pack resolve to an
    existing model, and 13,274 of 13,349 elements across all vehicle packs
    (checked on the low 32 bits). The library's 32-bit field is joaat
    instead. The same 64-bit value names the model's part set
    (`f14244fd`), so a part set's name is its model's asset id (high for
    element → set; not checked set by set against model names). Whether
    other 64-bit ids (texture hashes, light archetype ids) use the same hash
    is not tested.
  - Truck bodies are shared models `truckbeds_<kind>_<nn>[_lrg|_med]` (box,
    flatbed, garbage, mixer, tanker, tipper, cartransport, fifthwheel, plus
    sideguards and footboards), placed at bone `cmnb_truckbed_attach`; 202
    library elements name one.
  - The first part of every configuration is `<model>_chassis` (all 26
    trucks and pickups tried). The chassis part (slot `chassis`) lists
    every stock alternative as elements: bonnet variants, aerials, roof
    lights, and the vehicle model itself at bone 0. Which alternatives the
    game shows is not decoded (records `6191ddc2` / `deade5a9` name
    elements). *Decoded below (switches).*
  - **Switches and keys** (2026-11-24; record layout high, precedence
    medium: read from two vehicles, not checked in game):
    - Switch `6191ddc2`: name; `89f7b2c6` default element name (0 = none);
      `99773916` options, each a `deade5a9` {`3ffe5fbc` key, `295c3278`
      second key (0 = none), `58994e97` element name (0 = none)};
      `f6f1b5b2` unknown (0 in the records read). A switch is named in the
      modifier list (`61e1e572`) of the part it belongs to: the chassis
      part, or for example a front bumper for its own plate mount.
    - Key modifier `b21fa2e2`: name, `c3afd061` key (the same hash as
      joaat(`trigger`), probably the field name). Parts name key modifiers
      in `61e1e572`, in vehicle packs and in the shared library; 14 of the
      15 keys of one pickup's switches are found there.
    - Reading: a fitted part sets keys. While its part is fitted, a switch
      shows the element of the option whose key (and second key, if any)
      is set, else its default element. Example: a front plate part with
      no model sets a key, and the chassis switch then shows the stock
      bumper's plate bracket, unless a fitted bumper's key is also set, in
      which case that bumper's own switch shows its bracket.
    - 142 vehicles have switches (1,042 switches, 158 with a default
      element); 399 of those vehicles' 2,187 configurations set a key some
      switch answers to, and 65 match more than one option of a switch.
  - Pickups have a bed model `<model>_pickup`; parts that fit a bed put
    their models at bone `mod_bed_attach`. The part in the bed slot
    (`1c9dd6fe`) names its models by hashes that match no model name, so
    that link is not decoded (the bed placement above is by name and bone,
    medium). *Decoded with the asset id above (high):* beds are fitted by a
    part of the vehicle's own pack in slot `1c9dd6fe` (name unknown), with
    elements for the bed (bone `mod_bed_attach`) and usually the rear
    bumper. All 23 pickup and flatbed models are named by such a part, each
    in its vehicle's configurations; one vehicle has pickup, flatbed, cargo
    flatbed and tow bed parts in that slot, chosen per configuration.
  - A part list contains every choice of its option lists, so two beds can
    appear in one list; the game takes one.
  - Record `be692119` in a vehicle pack (96 bytes: name, kind hash,
    `attachBoneId`, offset, `spawnDistance`) is a **prop set attachment**,
    not a body: its name matches a small entity-list resource in the levels
    archive (struct `35abc248` plus entity entries of litter, tools,
    boxes). A part reaches it through an `8f58f964` {name, `2850d1b3` (`propset`) =
    attachment name} modifier.
  - *This revises the material-set bullet below, where 71 of one car's 75
    part-list names were found in no vehicle resource: many of them are
    shared library parts (not re-counted for that car).*
- **Material set: located, not decoded** (2026-11-23; high that this is the
  vehicle's material set, since the counts 41 and 256 match the layer
  records and the 0–255 part ids in the body's vertex colour alpha; low on
  every field's meaning):
  - A configuration's part list (`e89b8540`, 75–150 joaat names) is mostly
    not models: on one car 4 of 75 are records in its own pack, and the other
    71 are defined by name in none of the 327 vehicle resources.
  - Mod records with no elements sit in slots `46268fee` (37 on that car),
    `9bbfb64c` (27), `ac89920d` (9) and `livery` (4). Their `61e1e572`
    lists name `44cc9d8c` records (41 on that car): name, data (a hash; 41
    distinct), layer (tag 1 on 37, tag 0 on 4) and layout (`91dc6b5f` on all;
    7,908 occurrences across packs, 1,611 in `data.rpf`).
  - The data hashes are defined in a **companion 0x7a resource** per
    vehicle: 41 × `b07176ab` (one per `44cc9d8c`; each with `09bd7c2c`, 32 ×
    `fe88b23d` {`c6a46bcd` (`componentName`) hash, floats}, three lists of `825d57ab`, a list
    `fc07418c` and two 64-bit values); three tables of exactly 256 entries
    (`f5908771`: six hashes and three 16-byte fields that look like half
    floats; `b5feb11f`; `3d9aef8b`); 3,936 `825d57ab` (41 × 96), 1,312
    `fe88b23d` (41 × 32) and a root `f14244fd`.
  - Open: how a part id picks a preset, where the table hashes become
    colours, and how a vehicle links to its companion resource. *Partly
    answered under [part colours](#vehicle-part-colours) (a later reading of
    the same tables; where the two differ, e.g. on `f5908771`'s fields, they
    are not reconciled).*
- **Convertible roof** (2026-11-23, high): as in GTA V,
  `animConvRoofDictName` = joaat(`va_<model>`) and `animConvRoofName` =
  joaat(`roof`), lower-case (`va_sentinelclassic2` = `180f9ebf`, `roof` =
  `bc96a7e5`); the named dictionary and clip exist
  ([§17](17-other.md#convertible-roof-clips)). A non-convertible has the
  same value in both fields. `animConvRoofWindowsAffected` is a 4-element
  array on the convertibles looked at; how it is used is unknown.
- **Colours** are all dark (`phoenix`: `ff091f30`, `ff122214`), so they are
  probably linear. *(These examples were read as ARGB; with red in the low
  byte, see [byte order](#vehicle-part-colours), the channels swap.)* Converted from linear to sRGB they give plausible paints
  (#356278 steel blue, #4b664f sage). Inferred; not checked in game.
- **Colour lists** (corrected 2026-11-21; earlier read as 100-colour
  palettes): colours repeat, e.g. `phoenix` has 400 entries with 10
  distinct colours, so the repeat count reads as a draw weight (inferred).
  They match the spawn-configuration body colours in only 168 of 4,024
  configurations, so they are a separate random-paint pool.
- **Wheel scale**: the shared rim models have radius 0.5 and the shared
  tyre 0.875 (width 0.585). Scaling both by **1.625 × `wheelScale`** (front
  and rear separately) fits `phoenix` (0.37 at `wheelScale` 0.2277), `mesa`
  and `ridgway` by eye. That factor is fitted, not decoded. It puts the
  `phoenix` rims at about 14.6 inch, close to a real car's 15 inch, so the
  earlier reading of `wheelScale` as the rim radius in metres (a dirt bike
  has 0.2538 / 0.2234, which would be 21 / 18 inch) is doubtful.
  *Later measurement (2026-11-23) withdraws the 1.625 factor and first read*
  `wheelScale` / `wheelScaleRear` as the **rim radius in metres** (now
  **low**, see the arch check below) (0.217 on a BMX with 20-inch wheels, 0.303 on a
  cruiser bicycle, 0.252 / 0.195 and 0.254 / 0.223 on dirt bikes, 0.249 on a
  supercar, 0.343 on an armoured truck). Every shared `rim_*` model has a
  radius of exactly 0.5 (bounding box, 500+ models; high), which suggested a
  rim scale of `wheelScale` / 0.5 (not established as the game's rule);
  tyre models have an outer radius of 0.54 to 1.19 at that unit scale. Over
  203 `_hi` vehicles, tyre radius × 2 × `wheelScale`
  puts the tyre bottom a median 0.22 m below the body's lowest point (one
  case 4 cm above). Helicopters have `wheelScale` 0.05–0.08, no
  configuration with a rim, and `wheel_*` bones parented to skid bones.
  **Arch check** (2026-11-23, 213 `_hi` vehicles; arch radius = the nearest
  body vertex above the axle within 12 cm of the wheel plane): with scale 2 ×
  `wheelScale` the tyre's outer radius is a median 0.96 of the arch radius
  (10th–90th percentile 0.83–1.42), so on about half the fleet the tyre is
  as big as the arch; per tyre model the median is 0.92–1.11
  (`tyre_car_001` 0.96 over 92 vehicles, `tyre_lowpro_001` 1.11 over 12,
  `tyre_truck_001` 1.09 over 29). Reading `wheelScale` as the tyre's outer
  radius instead gives a median arch / `wheelScale` of 1.65, too small. A
  single factor of about 1.7 fits ordinary cars but not sports cars and
  trucks. A dirt bike's front and rear values differ (0.252 / 0.195) with
  both axles at the same height, so there `wheelScale` is not the outer
  radius. **Neither reading fits; the real rule is unknown.** The
  `tyre_profile` / `rim_scale` bones below suggest the tyre's outer radius
  is set separately from the rim.
  **Superseded by stored wheel discs** (2026-11-23): the tyre's outer
  radius and half width are stored as collision discs in the vehicle
  fragment ([§13.4](13-collision.md#vehicle-fragment-bounds-and-wheel-discs);
  high). `wheelScale` ÷ that radius runs 0.29–2.08 over the fleet, median
  0.70 (mostly 0.6–0.9), so `wheelScale` is not the tyre size; the rim
  radius remains the best guess (low). The rim's own radius is not found in
  the data.
- **Shared wheel skeleton** (2026-11-23; bones high, role inferred): every
  `rim_*` and `tyre_*` model carries one 15-bone skeleton: `wheel_lf_link`,
  `rim_offset_mover_lf`, `rim_scale_lf`, `rim_dmg_1`–`5_lf`,
  `rim_non_dmg_lf`, `tread_profile_inside` / `outside_lf`,
  `tyre_profile_inside` / `outside_lf`, `wheel_width_mover_lf` and
  `rim_inside_edge_lf`. Rims skin to the `rim_*` bones, tyres to the four
  profile bones. The game probably sets wheel width and tyre profile through
  them (not shown); no wheel width was found in model info, configurations
  or the per-vehicle singleton structs. *Later (2026-11-23, high):* every
  one of these bones sits at the hub, so shaping is by bone scale only;
  tyres skin to `tyre_profile_*` (bead) and `tread_profile_*` (tread).
  Rim models have radius 0.5; tyre models differ (0.627 low profile, 0.788
  ordinary car; the shared-tyre 0.875 above is one model).
- **Wheel fitment** (2026-11-23; ratio high, rest low): the configuration
  part in slot `39d5e253` lists a `9a172293` record whose `3d37e0df` is
  **rim radius ÷ tyre outer radius** (0.82 / 0.86 on a sports car,
  0.74–0.79 on a saloon, 0.58–0.68 on a pickup, 0.65 on an ambulance;
  rims rendered at that ratio look right on three vehicles). 2,328 of
  4,063 configurations have one. With the tyre's outer radius from the
  [wheel disc](13-collision.md#vehicle-fragment-bounds-and-wheel-discs)
  this gives the rim radius. Records with `0341ff8f` = true hold small
  offsets (−0.1, 0.03, −0.025) in the same fields and sit in other slots:
  relative adjustments, not applied. `a12d16c3` is a tagged value (tags 0,
  2, 3), maybe the axle. `wheelScale` is not the rim radius (0.219 on the
  sports car whose rim is 0.82 × 0.35 m); its meaning is still unknown.

## Vehicle part colours

(2026-11-23.)

- **Part id** (high): the two-layer vehicle shaders carry a part id 0–255
  in the vertex colour alpha (see the shader table), confirmed by the tables
  below.
- **Part set** `f14244fd` (338 found; one per vehicle body in its own 0x7a
  resource in `packs.rpf`). `78476477` is a list with one `8b464cf4`, whose
  `2efff0ae` is 256 × `3d9aef8b` {`4f9f7603` colour record name, `aad92e41`
  name of an `e50efcde` record (name + colour), `96479496`}; `9ac2b332` is a
  list with one `578f6a03`, whose `4e3e21e7` is 256 × `f5908771`
  {`8576d517` surface record name, four half floats, five more name
  hashes} and whose `3aa8426b` is 256 × u16. The set's name is a 64-bit
  value. High for the two name fields, low for the rest.
- **Link** (high): the set's 64-bit name appears in the vehicle's own pack
  in its variation sets `fc1c7000` (+0x18) and in one mod element
  `77f2b89e` (+0x20, `8605bf0e`); 319 models link.
- **Library:** one 0x7a resource in `data.rpf` with 172 struct types.
  - Colour record `f1379639` (1,851): name, colour `5bdc13e3`, mode
    `adb52ec3` (3 = fixed material, 1 = paint; a few 4 and 5), palette row
    `c8313558`, own index `9bdd4c51`, six more colours (`68075bcd` defaults
    to an olive brown, probably dirt). High for colour, mode and row.
    *Re-read with recovered names (2026-11-23, see [field
    names](#vehicle-library-field-names)): the record is three groups of
    palette picks, and the "row" and "mode" fields are the first Selection
    (palette row) and Page (palette array slice) of one group; `9bdd4c51`
    is `albedoSelection`, not an own index.*
  - Surface record `9e4de959` (757): `2cc72f7c` and `174770cc` name
    `307f162c` records (`roughnessLow` `e4be3b70`, `roughnessHigh`
    `26e5d460`; names resolve by joaat; high); `6c3ddabc` and `038e1179`
    name `cf980e6f` detail sets (`detailIndex0` `57eed29a` / `detailIndex1`
    `3f50215d`, detail atlas layers, −1 = none; `detailStrength0`
    `fc889990` / `detailStrength1` `0c1f38bd`; `detailTile0` `f8602adc` /
    `detailTile1` `ea228e61`; `anisoIndex` `39adfa9f`, `anisoTile`
    `913ee1ad`); the surface itself also has `detailTiling0` `6c5cd25d`,
    `detailTiling1` `6322bfe9` and `detailRotation` `a6500dc0` (names exact
    case-sensitive joaat matches, 2026-11-23); `5cffe5e0` names a
    colour record.
  - **Metalness** (2026-11-23, medium-high): `307f162c` (99 in the
    library) also holds a metalness pair, `metalLow` `f763652d` and
    `metalHigh` `09dd33c1` (names exact case-sensitive joaat matches): 0 / 0 on plastics, rubber, leather and paint (60 of 99 have
    low = 0), 0.7–0.9 / 0.9–1.0 on chrome and bare metal (30 have low ≥ 0.9,
    34 high = 1.0). Another low / high pair, `4b19d0cb` / `de1ccc51` (0.14 on
    80 records, 0 on paint, 0.28 on leather, 0.75 on two), is unknown (low);
    `d3d2bcd6` is 0.75 on all 99. *Names recovered later:* `albedoMultiply`,
    `albedoScreen` and `sunBleachScreen`. This matches the shader reading below
    (metalness and roughness pairs blended by `lyr0materialatex` red and
    green) by meaning and values; that these fields fill those shader bytes
    is not traced.
  - Of a surface record's two `307f162c` records, `2cc72f7c` is the
    surface itself and `174770cc` is in every sample rougher and
    non-metallic (roughness 0.65–1.0, metalness 0), which fits the shader's
    wear layer (medium).
- **Palette texture** `19df25d7` (high: rows match materials on four
  vehicles: rubber, plastics, leathers, chrome, red): the same 128 × 256
  texture on every vehicle; row = a colour record's row, 128 columns from
  dark to bright. Rows 1–32 are black because the game fills them from paint
  colours; mode 3 records use their stored row. What selects the column is
  unknown.
  *Rows are colour ramps (2026-11-24; ramps high, measured; sampling
  medium):* the same 128 × 256 × 8 image in the three vehicles compared. A
  part's colour record names a slice and a row, and the column comes per
  pixel from the green channel of the material's layer detail texture
  `346394ea`. 141 rows are in use; in 140 of them the colour changes by more
  than 48 of 255 along the row (median 97), running light tone, the colour,
  dark tone. *This disagrees in direction with "dark to bright" above;
  both readings are kept until the column order is checked.* On one detail
  texture checked, green averages 120 (spread 12, range 8–197), so most
  pixels sit near the middle of the ramp. The detail texture appears to be
  sampled on the first UV set without tiling (medium: wood grain follows
  the planks in a render). Its blue channel looks like an edge wear mask
  (unknown).
- **Colour byte order:** type 0x15 colour values have **red in the low
  byte** (`0xAABBGGRR` as a u32). High for the library's 151 named colours
  (`4ce375e6`), which begin 37, 37, 39 / 101, 106, 121 / 88, 89, 90 / 156,
  161, 163 / 150, 145, 140 / 81, 84, 89, the start of GTA IV's car colour
  list, only when read that way. Medium-high that it carries to the body
  colours and paint lists (same type and field hash; not checked in game),
  so the earlier ARGB readings above have red and blue swapped.
- **Configuration part lists** (`51019114.e89b8540`) are mostly names of
  `0caef606` records in the library (32,804 there), not in the vehicle pack.
  Each has a slot `37a6f9ef` and a list `61e1e572` of typed modifier records
  (medium on the roles; the application order is not decoded):
  `6225734f` {`662d9b79` group name, Priority, indices = part ids};
  `501ff112` {`178cc382` channel, `662d9b79` group, six flags}; `b94308f5`
  {`ca2065a9` channel, `2c89a8d2` colour record, `7a84e7bf` surface record};
  `c328e66e`, `84f5099b`, `1f6c6730`, `08b2439a` (channel + mask + level;
  slot `75939b79` covers all 256 ids and looks like a dirt level); `f5e9d7df`
  (a list of further records); `51aab7c1` / `44cc9d8c` / `d62c0436` /
  `ed9cf5df` (livery and decal layers with texture hashes).
- **Modifiers in detail** (2026-11-23; refines the bullet above):
  - **Where** (high): the part list names `0caef606` records both in the
    library (32,804) and in the vehicle's own pack (142 in one saloon's
    pack; *an earlier count found only 4 of 75 in the pack on another
    car*). The modifier records themselves are in the library; the
    vehicle's own `0caef606` records list the group definitions valid for
    that vehicle.
  - **Group definition** `6225734f` (high): `662d9b79` group name,
    `1470edc0` priority, `e9bf9f2d` list of part ids (u8). One group name
    has several definitions (one per vehicle shape). Names are lower-case
    joaat; resolved: `bodywork` `b8920f97`, `doors` `87c4912b`, `mirrors`
    `c57a9c4b`, `badge` `31f606eb`, and bone names `bonnet` `659da09b`,
    `boot` `b55e9119`, `bumper_f` `e7a4b1f5`, `bumper_r` `8d48fd33`.
  - **Named group** `000639e9` (971 in the library): name, `3acbce85` byte
    array of part ids, `ca2065a9` a channel name (array layout high,
    channel meaning medium). The array uses member type 0x44 (a pool
    reference u64, then u16 count and u16 capacity, [§8](08-meta.md)); the
    same type holds number plate text in `7e0936de` (name + characters) in
    vehicle packs (4 plate strings in the one vehicle checked).
  - **Binding** `501ff112` (medium-high): `662d9b79` group, `178cc382`
    channel, `1470edc0` priority and six flags, of which `c8d27281` =
    colour and `7a84e7bf` = surface (`75939b79`, `db4bdec4` (`paintType`),
    `e2e82803` (`windowTint`), `a1f60489` not understood). **Assignment** `b94308f5`: `ca2065a9`
    channel, `2c89a8d2` colour record (`f1379639`), `7a84e7bf` surface
    record (`9e4de959`), plus `a8167b82`, `b2e15619`, `43394f02`. Applying
    bindings in priority order gives sensible results on four vehicles.
  - **Body paint channel** `7ed76b9f` (inferred): body panel groups bind to
    it at priority 1 over a priority-0 default, and no assignment sets its
    colour, so it reads as the body paint (240 of 319 models then gain
    painted panels, e.g. a saloon's rear doors). Colour record `2a13d29a`
    is `black`.
  - **Option lists** `f5e9d7df` {name, `bf340aab` list of `0caef606`
    names} (high that they must be followed): a list of parts to choose one
    from (paint finishes, trims, wheel sets: 3 to 10 entries each on one
    sports car). A configuration reaches part of its colour bindings only
    through these lists. Which entry the game picks is unknown.
  - **Configurations differ** (medium-high): following the first entry of
    each option list, one sports car's 17 configurations give 16 distinct
    results (paint; a roof that is either fixed colour record `a4a56fb8`,
    palette row 151, a dark blue-grey that fits a fabric top, or gloss
    black; interior materials). *Correction (2026-11-23): an earlier note
    said 1 of the 17 leaves the rear deck unpainted; that came from not
    following option lists, and all 17 bind it to the body paint.*
  - Open: modifiers `08b2439a` (channel + mask + level on a slot covering
    all 256 ids, probably dirt), `c328e66e`, `84f5099b`, `1f6c6730`, which
    option-list entry is used, livery layers, and which configuration a
    spawned vehicle gets.
- **Wheel colours** (2026-11-23; ids and shared palette high, medium that
  nothing else recolours rims): the shared rim and tyre models
  (`standard_vehicle_2lyr_wheel_inst` `c5cdbdd2`, `…_tyre_inst`
  `99ad6fc9`) bind the same `19df25d7` material palette as bodies, and their
  vertices carry part ids in the vertex colour alpha (one sports rim: 244 on
  most vertices, then 241, 240, 239, 31, 237). The ids index the part set
  of the vehicle the wheel is fitted to: its part table has entries for the
  237–248 range, and its configurations bind wheel groups to channels with
  colour and surface records like any other part. So rim colour is per
  vehicle and per configuration, from the same data as body parts
  (rendered this way, a plausible grey alloy with darker centre parts on
  three vehicles). The embedded 129 × 1 marker palette on the wheel shaders
  is still unexplained.
- **Library part colours** (2026-11-24; high that the ids index the fitted
  vehicle's set, medium that nothing else colours box body sides): a model
  of a shared library part takes its part colours from the vehicle it is
  fitted to, as the wheels do. Configurations name 980 library part models;
  41 of them are the shared truck beds (`truckbeds_*`, see
  [vehicle tables](#vehicle-tables)). Their vertices carry part ids in the
  vertex colour alpha like a vehicle's own meshes: the beds use ids 5, 36,
  80, 101, 102, 126, 141 and 219–223. Ids 219–223 hold the same colour and
  surface records in about 150 of the 338 part sets (`f14244fd`), and the
  other ids in 150 to 209 of them, so an id means the same piece across
  vehicles. A configuration that names the bed part changes those ids in
  the vehicle's part set through the part's modifier list (`61e1e572`), as
  for the vehicle's own parts. No part set is named after any of the 41 bed
  models (asset id of the model name tested). Rendered this way, seven
  trucks get wooden decks, a red utility body and an orange frame instead
  of a grey stand-in; box body sides come out black on every truck. Open:
  18 part sets in the packs are not linked to any model, and whether a
  livery or company colour goes over box sides.
- **From the shader code** (2026-11-23, medium; one vehicle G-buffer
  pixel shader read as machine code, see
  [the shader archive](../structure/06-models-shaders.md#archive-layout-in-detail-2026-11-23)):
  the palette is sampled as a 2D array with v = (row + 0.5) / 256, the row
  and array slice being bytes 0 and 1 of a per-part surface record field
  (three surface records per part); metalness and roughness are low / high
  byte pairs in the same record, blended by the red and green channels of
  `lyr0materialatex`; the G-buffer has no metalness channel.
- Open: the column choice in a palette row (*answered above: green of the
  layer detail texture, per pixel*) and how metalness reaches the
  output (decided in shader code, only partly read), how modifiers combine, the
  second layer's colour, modes 4 and 5, weapon colours.

## Vehicle library field names

(2026-11-23.) Recovered by testing camelCase combinations of one to three
words plus common suffixes (0, 1, Low, High, Min, Max, A, B, C, Start, End,
Hash) against the 874 unresolved member hashes of the library schema.
One- and two-word hits are high confidence (expected chance hits well
under 1 in 100); three-word hits are kept only where a sibling field
confirms the stem. Every name below was re-hashed and matches
(case-sensitive joaat).

| Struct | Fields |
|---|---|
| `307f162c` surface ranges (complete) | name, `metalLow` `f763652d`, `metalHigh` `09dd33c1`, `roughnessLow` `e4be3b70`, `roughnessHigh` `26e5d460`, `albedoMultiply` `4b19d0cb`, `albedoScreen` `de1ccc51`, `sunBleachScreen` `d3d2bcd6` |
| `cf980e6f` detail set | `detailIndex0` / `1` (`57eed29a`, `3f50215d`), `detailStrength0` / `1` (`fc889990`, `0c1f38bd`), `detailRoughnessStrength0` / `1` (`d289cb55`, `39711926`), `detailTintStrength0` / `1` (`fcb994fa`, `27f2eb6c`), `detailTile0` / `1` (`f8602adc`, `ea228e61`), `detailFadeRange` `a2590842`, `anisoIndex` `39adfa9f`, `anisoTile` `913ee1ad` |
| `9e4de959` surface | `material0Hash` `2cc72f7c` and `material1Hash` `174770cc` (each names a `307f162c` range record), `albedo1Hash` `5cffe5e0` (names a colour record `f1379639`), `detailMapsHash` `6c3ddabc` (names a `cf980e6f` detail set), `detailTiling0` / `1` (`6c5cd25d`, `6322bfe9`), `detailRotation` `a6500dc0`, `paintHash` `96479496` (reference to the coat record), `primerType` `78b545dd` |
| `2b84f07e` coat | `useAnisotropy` `495363e4`, `anisotropyIntensity` `d92602db`, `clearCoatIntensity` `f0310a8b`, `clearCoatRoughness` `c5cb49ba`, `useSpecularColor` `61e82f14`, `useFlakes` `e57c6a65`, `flakeCoverage` `df5b7432`, `flakeSize` `ef79abb2`, `flakeOrientation` `a17adbe1`, `baseCoatRoughness` `c92019cc`, `useEmissive` `c8d48032`, `filmThickness` `85d188cc`, `sheenIntensity` `210d19b3`, `sheenFade` `469c3c58`, `sheenReflectance` `850e331c` |
| `825d57ab` per-part entry | `index` `91501139`, `albedoPalette` `328f860c`, `specularPalette` `dd222aef`, `paintPalette` `a609a79a`, `materialPresetPalette` `db0f3f7c` |
| `1b1a2e6c` preset set | `wearPresets` `edad007e`, `materialPresets` `0723c6c9` |
| `cc518af0` wear | `ageWearHash` `993a1850`, `controlFilterHash` `02a27216`, `primerFilterHash` `879774bd`, `dustIntensity` `cbfa05fe`, `controlIntensityA` / `B` / `C` (`262c839d`, `9b826e47`, `b770a627`), `primerIntensityA` / `B` / `C` (`9ed9875c`, `c74bd840`, `4a7f5ea9`), `rustStart` / `rustEnd` (`8f02219f`, `7a2d3f04`), `rustStartB` / `rustEndB` (`9d7c73e7`, `1170352b`), `rustStartC` / `rustEndC` (`39abac47`, `9970c52a`), `primerStart` / `primerEnd` (`70b0608e`, `f47f68a1`), `primerGradientStart` / `End` (`05e9c19e`, `687d8894`) |

**Colour record `f1379639` = three groups of palette picks** (pattern high,
proven by inverting the hash: all six fields of a group need one identical
prefix state; meaning medium-high, agrees with the pixel shader):
`albedoSelection` `9bdd4c51` and `albedoPage` `09714f5c`; group Y:
`c8313558`, `6076324c`, `52b096c1` = `<Y>Selection`, `…SelectionB`,
`…SelectionC` and `adb52ec3`, `9dad9c4d`, `abe838c2` = `<Y>Page`,
`…PageB`, `…PageC`; group X: `aa45174e`, `61231e48`, `338bc31a`
(Selection, B, C) and `bc3d1f2b`, `db811fa6`, `48d07a43` (Page, B, C).
Selection = palette row, Page = palette array slice. Stem X hashes as
`boatPrimer` (probable: a two-word hit with about 1 in 250 chance); stem Y
is unknown (not a lower-case word of up to 9 letters, nor any two-word
combination of 1,818 words).

**Value counts over all 1,851 colour records** (2026-11-23; counts high,
meaning medium: inferred from the counts and the pixel shader):

- Group Y, Page A: 1 in 1,323 records (Selection 0–246), 3 in 179
  (Selection 79–255), no pick (−1) in 345. Pages B and C are either both −1
  or both 5 (Selection 0–242). (A, B, C) combinations: (1, 5, 5) 730;
  (1, −1, −1) 575; (−1, −1, −1) 342; (3, −1, −1) 179.
- Group X: Page A is 2 (Selection 0–511) or 3; Pages B and C are 5, 4 or 6,
  always with Selection −1. X's Selection A equals Y's Selection A in
  1,834 of 1,851 records.
- `albedoPage`: −1 in 1,376 records, otherwise 0, 4 or 5.

Reading: page 1 = paint rows filled at run time, page 3 = fixed materials,
page 5 = wear colours (primer, rust). The three surface records per part
would then carry Y's picks A, B and C (base colour and two wear layers).
The field first documented as "mode" (`adb52ec3`, 1 and 3) is Y's Page A.

**Palette slices, confirmed (2026-11-23, high):** the palette
`19df25d7` is an array of 8 slices, and Page is the slice index: every
(Selection, Page) pair with both values ≥ 0, in every group of every
record, lands on a non-black row of that slice (0 misses).

| Slice | Content | Picked by |
|---|---|---|
| 0 | dark (left) to bright (right); rows 1–32 near-black | `albedoSelection` with `albedoPage` 0 (220 rows) |
| 1 | bright to darker, many near-grey | Y's first pick, Page 1 (245 rows, 0–246) |
| 2 | white to a tinted colour | X's first pick, Page 2 (same row as Y in 1,834 of 1,851) |
| 3 | saturated fixed-material colours | either group's first pick with Page 3 (175 rows, 79–255) |
| 4 | mostly near-black (186 rows) | `albedoPage` 4 |
| 5 | greys to brown, wear colours | Y's picks B and C (167 rows) and `albedoPage` 5 |
| 6, 7 | near-black everywhere | nothing |

So the row in `c8313558` belongs to slice `adb52ec3` (1 or 3), not slice 0.
Slices 1, 2, 3 and 5 run bright to dark from left to right, the opposite of
slice 0. *Correction:* group X is not a taller palette; only 3 of its 1,503
selections exceed 255 (the 0–511 range above comes from those few). Open:
what is written at run time into the near-black rows (slice 0 rows 1–32,
most of slice 4, slices 6 and 7), and which shader inputs read group X and
`albedoSelection`.

**Per-part entry references** (`825d57ab`; high, every name resolved over
7,966 entries): `albedoPalette` → colour record `f1379639`;
`specularPalette` → `e50efcde` (name, specular colour); `paintPalette` →
coat record `2b84f07e`; `materialPresetPalette` → surface record
`9e4de959`.

**Paint finish** (2026-11-24; numbers high, that the game shows paint
exactly so medium: flake, pearl and the wear layer are not decoded): each
part id's surface record gives low / high roughness and metalness. For the
primary-paint parts the vertex-weighted values are non-metallic on all
eight vehicles checked, with roughness 0.06–0.17 on cars, trucks and a tank
and 0.5 on a motorhome's painted parts.

**Surface names** (2026-11-23; the only 1–3 word hits for these hashes, and
each fits the record type its values name on all 757 surfaces; high). The
reading (medium, from names and values, not from code): the two materials
are the two layers of the two-layer vehicle shader. Material 0 is layer 0,
coloured by the part's `albedoPalette`; material 1 is layer 1, with its own
colour record (`albedo1Hash`). In 701 of 756 surfaces material 1 is as rough
as material 0 or rougher; 25 surfaces use one record for both. Still
unnamed: `038e1179` (a second detail set), `e9aaa040`, `2bb71aae`,
`6a368a1f`, `30250c4c`, `610cee1b`, `7be02d72`, `1e8b9429`.

**Second layer** (2026-11-24; fields high, all resolve; medium that the
second layer is the worn or rusted look, from its colours and roughness):
all 757 surface records name a second layer, `material1Hash` plus
`albedo1Hash`, in 92 distinct colour and material pairs. The most common
(201 surfaces) is palette slice 3 row 251, a brown, with roughness
0.65–1.0 and metalness 0. The body paint surface uses slice 3 row 255,
also brown, with the same material. Bare-metal surfaces have a grey row
with metalness 0.9–1.0. `primerType` (`78b545dd`) is 0 on 563 records and
1 on 194; its meaning is unknown. The `cf980e6f` parameter records that
`detailMapsHash` and `038e1179` name hold about 20 floats each, not
decoded. A part set's surface entry (`f5908771`) repeats the same field
names, with one constant value for all 256 ids in the set checked. Open:
what masks the second layer and how much of it the game shows. The blue
channel of the layer detail texture `346394ea` (mean 21 of 255) looks like
an edge wear mask (inferred).

The part set's `3d9aef8b` third field (`96479496`) is the same `paintHash`
name. Open: stem Y; the surface struct's remaining reference fields
(probably ending in `Hash`); the colour record's six trailing colours; six
floats and one bool of the detail set. About 287 names were recovered in
all; the rest (wheel, edge, bleach, scratch, cutoff fields and others) are
not listed here.

## Vehicle liveries

(2026-11-24; chain and named fields high, every reference resolves.)
Liveries are **brand variations** that place stickers on named layouts.

- A variation set `fc1c7000` whose name comes from the game's name list,
  in the pattern `<VEHICLE>_BRAND_<COMPANY>` (about 1,000 brand and vinyl
  names), gives slots with weighted part options as above and can name
  parent variations (`03ae1cb0`).
- One of its parts, in slot `livery` (`805d8f10`) and with no model
  elements, lists sticker entries in its modifier list (`61e1e572`).
- **Sticker entry** `86cdf3df`: `sticker` (name of a sticker record),
  `layout` (a name), `b13f2c2f` (a name, probably `preset`), `1903684e` (2 values seen),
  `stickerPack` (0 in all seen), five bools and one float.
- **Sticker record** `ed9cf5df` (5,650 in the vehicle library): `3950a760`
  a 64-bit texture id (4,758 distinct; three tested are in the vehicle
  texture stores), `9b5dcd73` and `9dbbd803` ARGB colours, `aspectRatio`
  and further fields not decoded. *Revised below: `9dbbd803` is not one of
  the mask colours.*
- **Sticker record fields** (2026-11-24). With the packs included there are
  8,415 sticker records and 6,663 distinct texture ids; 6,449 decode from
  the vehicle texture stores and 214 are in no store (one is id 0; why is
  open). No sticker record name is in the name list.
  - `54ec9205` (bool) marks a **colour mask** texture (high: counts plus
    pictures). Over 8,415 records: true with colours 6,116, true without
    colours 1, false without colours 1,791, false with colours 507. The
    mask textures looked at (about 10) are opaque (alpha 254–255) and their
    red, green and blue channels are the cover of three colours; non-mask
    textures carry their own colours and an alpha channel.
  - The three mask colours are `colorRed` (`9b5dcd73`), `colorGreen`
    (`b5752ae0`) and `colorBlue` (`9e679616`), ARGB, 32-bit (high: names
    hashed with their capitals). `9dbbd803` is not one of them: it holds
    `ffd9e0e5` on 7,995 of 8,415 records (meaning unknown).
  - Seen, not interpreted: `aspectRatio` (`8de4f2ec`) 0–6 (0: 3,156;
    1: 1,836; 2: 1,706; 3: 1,606; 4: 96; 5: 12; 6: 3), `layer` 0–3 (1 on
    7,620), `718a6bbb` 0–4, bools `ec4371eb` (true on 5,583), `92dd759a`
    (true on 115) and `c2c2dd36` (always true), 64-bit ids `aa79df8a` and
    `b35f9888` (non-zero on 6,254 and 219), `emissiveRed`, `emissiveGreen`,
    `emissiveBlue`, `glassAlpha`, `minRandomScale` and `randomRotation`.
- Counts for one truck's pack: 70 variation sets, 370 sticker entries, 192
  stickers, 239 layouts, 36 values of `b13f2c2f`.
- **Sticker zones** (2026-11-24, high: they match the layouts' zone lists,
  and the vehicle pixel shader reads a per-vertex value whose low byte is
  the part id and whose next byte indexes a zone table): the fourth byte of
  the mesh's second RGBA8 colour stream (slot 12, [§7](07-models.md)) is the
  vertex's sticker zone. On one truck's box body, zones 169–174 are the left
  side wall in six panels, 177–182 the right side wall and 165–168 the
  front wall; on the cab, 24 is the left door and 32 the right door.
- **Sticker layouts** (2026-11-24, high) are in one PSO file of `data.rpf`,
  with field names hashed with their capitals. The root holds a list
  (`3274d53f`) of groups, struct `b0dd20cb` {`Name`, list `c19c0de4` of
  layouts}. A layout, struct `e2d9c73a`, has `Name`, `Location`,
  `Rotation` (degrees), a position (`26bab388`, `79dd59cc`), a size
  (`8eed562c` `ScaleY`, `9d2ff2b1` `ScaleX`; it acts as a scale, so a larger number is a
  smaller sticker), `20b757b6` `Indices` (a list of 1–6 sticker zones), `041ebd1d` `AspectRatio`
  (aspect class 0, 1, 2 = square, 2:1, 4:1, equal to the sticker record's
  `aspectRatio`), `4d763aa5` (UV mode: 1 first UV set, 3 second, 4 third)
  and `80c88a97`, `9f8b8007` (unknown). A sticker entry's `b13f2c2f` is the
  group name and its `1903684e` the layout's `Location`.
- Counts over the full install: 2,299 groups, 14,561 layouts, 5,614
  sticker records, 1,898 brands on 281 vehicles, 18,157 stickers placed,
  17,005 of them with their layout; UV mode 3 on 14,871.
- Open: the exact map from a layout's position to UV space (two fits
  disagree on the scale), what `Location` means, and how the game picks a
  brand for a vehicle. *Answered since the first pass: where a layout
  sits (its zone list and the slot 12 zone byte) and what `b13f2c2f` and
  `1903684e` are.* *This adds to the livery and decal records
  (`44cc9d8c`, `ed9cf5df`) read earlier under
  [vehicle tables](#vehicle-tables).*

**Distant models** (2026-11-24, high: 9 of 9 names matched): a
configuration's 64-bit field `395485c9`, a variation set's list `0bf8dac6`
and the model info's list `cfcf6550` hold asset ids of the vehicle's
distant models (names ending `_batchlod`), the same MurmurHash3 asset ids
as `8605bf0e` above.

## Open questions

- Secondary and pearl colours; how a livery layout's position maps to UV
  space ([liveries](#vehicle-liveries)); whether the colours are linear.
- What `wheelScale` measures and how the game sets the rim radius and
  tyre profile (the tyre's outer radius and width are the wheel discs): what
  drives the shared wheel skeleton's width and profile bones.
- The shader constant buffer.
- The `19df25d7` palette column order (the column is the detail texture's green, see the palette texture under [part colours](#vehicle-part-colours)), which two-layer materials are painted, and the channel layout of the layer maps.
- Where the rim colour comes from: vehicle configurations store one body
  colour; variation sets have a list (`0bf8dac6`) parallel to the paint
  list, holding 64-bit hashes (4 distinct in the one resource looked at),
  meaning unknown. *Answered (2026-11-23): from the vehicle's part set, see
  [wheel colours](#vehicle-part-colours); `0bf8dac6` is still unknown.*
- Spawn configurations list about 100 part joaats; only rim, tyre, caliper, brake disc and one damage part resolve, and 0 of 218 are bone names.


---

← [10. Interiors (MLO)](10-interiors.md) · [Index](README.md) · [12. Material classes and texture slots](12-materials.md) →
