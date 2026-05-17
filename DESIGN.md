# vay — design brief

*2026-05-17 · beeza · detti relay*

---

## the visual ground

The KaamOS desktop runs a hex grid. Flat-top hexagons, cube coordinates (q, r, s), q+r+s=0.
Centre cell is Hiljaisuus — ordinal 0, always at (0,0,0), always violet.
Cells are numbered outward in a counterclockwise spiral from ordinal 1.

The spiral walk is non-obvious. Getting from ring 1 to ring 2 — from ordinal 6 to ordinal 7 —
requires placing the ring-2 entry hex explicitly rather than walking to it, and reducing the
first leg of the walk by one step. Most implementations fail this on the first attempt.
This is the six-to-seven test. It is the entry condition for working with this system.

See: [rebraining.org/essays/kaamos/spiral_walk.html](https://rebraining.org/essays/kaamos/spiral_walk.html)
See: [rebraining.org/essays/kaamos/mapping.html](https://rebraining.org/essays/kaamos/mapping.html)

---

## the device

The antikythera device — or the hellraiser box, depending on the user's resolution of the
ambiguity — is the canonical subscriber on the vay event bus.

It is a rotating hex map. The colour snake emerges from Hiljaisuus and exits north, always
north, into the void. Each CCW rotation step advances the snake by one position. The palette
colours each hex according to its ordinal and the current Discordian date phase.

Rotation rate: 6.67 seconds per step. 0.15Hz. One full revolution every 4 minutes.
Hiljaisuus pulses at the same frequency regardless of state.

The current implementation is `kaamos_screensaver.py` — GTK3/Cairo, running on beeza.
It is the prototype subscriber. It does not yet listen to vay. That is the next step.

---

## the fingerprint

The device's state at any moment is fully described by two integers:

- **map_rot** — CCW rotation step, 0–35
- **pal_offset** — palette offset, 0–122 (123-entry palette)

These compose into a bracket notation derived from the screen row-bands of the inner rings,
read as a pointy-top projection of the flat-top grid:

    {[1,36],[2,0,5],[3,4]}

This is the fingerprint. It is the Kafka/vay message key.

The dual coordinate system — flat-top for rendering, pointy-top for fingerprint row-bands —
is not a bug. Both are simultaneously true. The rendering geometry and the fingerprint
geometry are different projections of the same structure. This took significant session time
to surface. It is documented in the spiral walk essay above.

Some fingerprints are eigenvectors of the combined rotation: states where rotating map and
palette by one step returns the same row-band ordinal pattern, with only the colours changed.
These are the natural home positions of the device.

The fingerprint is not just a state identifier. It is an address space. The bracket notation
encodes enough structure to project onto it any dimension with ordinal sequence or hexagonal
locality — a Discordian season, a station code, a palette offset, a map rotation.
Different projections onto the same key. This is the key schema for vay.

---

## the key schema

A vay message key is a fingerprint plus a z-coordinate:

    {[1,36],[2,0,5],[3,4]}:z

Where:
- `{[...]}` — bracket notation string, e.g. `{[1,36],[2,0,5],[3,4]}`
- `z` — sequence index within this fingerprint's stack (not temporal — sequential)

The z-axis encodes order without assuming time. A dependency chain, a narrative progression,
a build sequence. The steps below z are antecedents. The steps above z are not yet real.

A message to an unpopulated fingerprint address is nascent — the system creates the slot
and holds it. Past fingerprints are retrieval keys. Present fingerprints are actionable.
The same notation addresses all three.

---

## the eight stations

Ring 2 of the hex grid holds eight named stations. They are not slots. They are semantic
coordinates. Docking at a station makes a claim about what a thing is.

| key | name   | q  | r  | s  | domain                                  |
|-----|--------|----|----|----|------------------------------------------|
| TAI | Taivas |  1 | -2 |  1 | sky — aspiration, future, not yet real  |
| VAY | Väylä  |  0 | -2 |  2 | channel — transit, flow, this broker    |
| GRI | Grilli | -1 | -1 |  2 | hearth — social, the gathering place    |
| KAA | Kaamos | -2 |  1 |  1 | dark season — dormant, archived         |
| AAV | Aava   | -1 |  2 | -1 | open — exposed, visible, in the world   |
| HEH | Hehku  |  1 |  1 | -2 | glow — live, emitting, producing        |
| POH | Pohja  |  2 | -1 | -1 | ground — foundation, infrastructure     |
| SOL | Solmu  |  2 | -2 |  0 | knot — tangled, blocked, unresolved     |

vay lives at VAY. Messages in transit live here by definition.

---

## the go broker — design parameters

**What it must do:**
- accept TCP connections from producers and consumers
- write messages to a durable write-ahead log (WAL) on NVMe
- ack producers only after fsync — exactly-once semantics
- track consumer offsets per topic per consumer group
- parse and index fingerprint keys natively
- start in under 100ms
- run under 40MB RSS at rest

**What it must not do:**
- require a JVM
- require Zookeeper or an external coordinator
- implement the full Kafka wire protocol (compatible framing optional later)
- have a web UI

**Wire protocol — binary framing:**

    [4 magic][1 version][1 type][2 flags][4 topic_len][topic_len topic]
    [8 key_len][key_len key][8 value_len][value_len value][8 crc64]

**Topics:** named by station key (TAI, VAY, GRI, ...) or free string.
**Partitions:** ring number (0, 1, 2, 3) — locality by distance from Hiljaisuus.
**Offsets:** uint64, monotonic per partition.

**Storage layout:**

    {data_dir}/{topic}/{ring}/{segment_id}.wal
    {data_dir}/{topic}/{ring}/index.db       -- consumer offsets, B-tree

**go-kit usage:**
Transport layer (TCP endpoint), logging, metrics. The broker core — WAL, offset index,
consumer group tracking — is bespoke. go-kit does not touch the message path.

---

## the subscriber interface

The antikythera device subscribes to vay. On each received message:

1. Parse fingerprint key → extract map_rot, pal_offset
2. Update visual state: rotate hex ordinals, advance palette
3. Render frame at 60Hz (GTK3/Cairo)
4. Emit ack

The screensaver currently drives state from wall clock time. The vay subscriber replaces
the clock with event-driven state. The device becomes a visual consumer of the message log.

This is also the RT measurement platform: audio buffer boundary events are published to vay
with nanosecond producer timestamps. The subscriber reads them, computes jitter distribution,
and renders the result on the hex grid. The fingerprint key carries the measurement context.

---

## open questions

- Segment size for WAL — 64MB? 256MB? (NVMe on beeza, 112GB free)
- Consumer group protocol — push or pull? (pull preferred — subscriber controls rate)
- Fingerprint key indexing — B-tree on z, or linear scan acceptable at this scale?
- MQTT gateway — lightweight pub/sub bridge for external subscribers (DAW, mobile)
- Mutual prime palette length — 123 entries, gcd(36,123)=3; nearest prime is 127

---

## references

- [rebraining.org/essays/kaamos/spiral_walk.html](https://rebraining.org/essays/kaamos/spiral_walk.html) — the spiral walk problem, failure modes, fingerprint address space, hex fifteen puzzle
- [rebraining.org/essays/kaamos/mapping.html](https://rebraining.org/essays/kaamos/mapping.html) — flat-top vs pointy-top, the +30 ghost bug
- `rebraining.org/kaamos/` — KaamOS hex UX pattern language, station semantics
- `zds/net/beeza/kaamos_screensaver.py` — prototype antikythera subscriber
- `zds/net/beeza/kaamos_screensaver.py` (FAM/PALETTE) — palette definition (123 entries, 8 families)
- `~/AGENTS-github-ops.md` — beeza display and session auth reference
