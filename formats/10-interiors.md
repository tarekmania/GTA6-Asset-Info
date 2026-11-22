# 10. Interiors (MLO)

*Part of the [SixNG format specification](README.md). Numbers as of 2026-11-21; confidence: confirmed / inferred / unknown.*

The struct hashes in this section are the JOAAT of the GTA V names given
([§8](08-meta.md)).

| Struct | GTA V equivalent | Fields | Confidence |
|---|---|---|---|
| `09cc28be` (592 B, 2,590 instances) | `CMloInstanceDef` | entity prefix; archetype at +0x44. **Rotation stored as is, not inverted** (unlike entities) | confirmed |
| `10506455` (544 B, 1,706) | `CMloArchetypeDef` | +0x74 name, +0x40 / +0x4C bbox, **+0xC0 entity pointer array** (into the same file's entity block), +0xD0 rooms (`0b1811f1`), +0xE0 portals (`995072ca`), **+0xF0 entity sets** (`d6a799f9`, entity pointers at +0x18) | confirmed |

All 2,590 instances resolve; 2,241 have entities (453 K entity
placements).

## Interior contents (confirmed)

Interior dressing is not in the world map files. It sits in 961 **interior
archetype files**: files with one or more `10506455` and no map data
(`d3593fa6`). Their entity list (`ce501483`) is the interiors' contents in
**interior-local space** (bbox around the origin); the world places them
only through interior instances (`09cc28be`). Read as world positions, these
entities show up as unparented HD entities near (0, 0, 0).

| Struct | Fields (GTA V names) |
|---|---|
| room `0b1811f1` (`CMloRoomDef`) | name (+0), bbMin / bbMax, blend, timecycleName, flags, portalCount, floorId, exteriorVisibiltyDepth, **attachedObjects** (u32 indices into the interior's entity list), reflectionProbes |
| entity set `d6a799f9` (`CMloEntitySet`) | name (+0), locations (room indices), entities (+0x18) |
| portal `995072ca` (`CMloPortalDef`) | roomFrom, roomTo, flags, mirrorPriority, opacity, audioOcclusion, corners, attachedObjects |

Room `e69e4c84` is `limbo` (room 0, outside), as in GTA V.

**Room flags** (inferred). The room `flags` field is a bitfield. GTA V uses
ten bits; whether GTA 6 keeps the same layout is not confirmed against real
room values yet, so treat this as a starting point to check:

| Bit | GTA V meaning |
|---|---|
| `0x001` | freeze vehicles |
| `0x002` | freeze peds |
| `0x004` | no directional light |
| `0x008` | no exterior lights |
| `0x010` | force freeze |
| `0x020` | reduce cars |
| `0x040` | reduce peds |
| `0x080` | force directional light on |
| `0x100` | don't render exterior |
| `0x200` | mirror potentially visible |

These are the public GTA V MLO room-flag meanings; no game files are
involved. A quick check is whether observed `flags` values stay within
`0x3ff` and whether rooms known to dim exterior light set `0x004` / `0x008`.

So every map file has a **space**:

| Space | Files | Entities |
|---|---|---|
| world | 23,528 | 1,669,203 |
| interior-local | 961 | 540,665: 394,112 in a room, 139,497 in an entity set, 540,447 named by an interior |
| group-local (prefab groups `35abc248`, [§9.3](09-placements.md)) | 6,296 | 39,622 |
| mixed | 0 | — |

7.5 % of room members resolve to a room name (Warehouse, Office, Shop,
GARAGE, storage, main, limbo).

**Interior collision.** Interior bounds are in interior space and carry no
name; the archetype's physics dictionary is its own name hash, so nothing
matches by name. Links are made by:

| Method | Rule | Share | Confidence |
|---|---|---|---|
| exact | bounds bbox = interior bbox within 0.25 m | 305 | high |
| ordered | the archive stores interiors and their bounds in the same order; between neighbouring exact anchors, best bbox IoU > 0.3 | 899 (~91 % right on hold-out) | medium |
| floor fit | share of the interior's entities standing on an up-facing triangle of the candidate (0.3 m below to 1.5 m above) + 0.3 × IoU | part of 463 | high/medium/low by margin |
| bbox | best IoU over all interior-space bounds | rest | mostly low |

Total 1,667 of 1,704 interior archetypes (97.8 %) linked, 79.3 % at high or
medium confidence.

## Open questions

- Entity-set selection rules; most room names; interior archetype names
  (they are not in the archetype tables).
- A direct (non-geometric) link between interiors and their bounds.


---

← [9. Placements, LOD links, archetypes and batches](09-placements.md) · [Index](README.md) · [11. Vehicle paint and vehicle tables](11-vehicle-paint.md) →
