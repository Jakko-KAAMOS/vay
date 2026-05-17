# vay — forth interpreter brief

*2026-05-17 · addendum to DESIGN.md*

---

## what and why

A Forth dialect — working name **kaamos-forth** — is bundled with vay.
It is the command language for the antikythera device and the primary interface
for directing hex map graphics from a terminal, a script, or an AI session.

Forth was chosen because:
- stack-based evaluation maps naturally to hex grid traversal (push coord, operate, pop result)
- words are composable: a gesture vocabulary builds from primitives
- no parser — tokenise on whitespace, execute immediately
- tiny: a usable interpreter is under 1000 lines of Go
- it can be embedded in the vay broker as a scripting layer without a separate process

---

## implementation language

Go. Embedded in the vay binary. No cgo, no external dependencies.
The interpreter runs in the same process as the broker.
A vay client can send Forth source over TCP; the broker evaluates and emits
the resulting vay messages. The screensaver never sees Forth — it sees messages.

---

## the stack model

The data stack holds three types of value:

    INT    — rotation step, palette offset, z-index, ring number
    COORD  — cube coordinate triple (q r s) as a single stack value
    FP     — fingerprint string e.g. {[1,36],[2,0,5],[3,4]}

Words operate on these types. Type errors halt with a stack trace.

---

## primitive words

### navigation
    NORTH  SOUTH  NE  NW  SE  SW   -- step one hex in direction from top-of-stack COORD
    RING   ( coord -- int )        -- ring number of coord
    ORD    ( coord -- int )        -- ordinal of coord in spiral
    COORD  ( int -- coord )        -- ordinal to coord

### rotation
    ROT+   ( n -- )   -- rotate map CCW by n steps, emit STATE message
    ROT-   ( n -- )   -- rotate map CW by n steps
    TICK   ( -- )     -- emit one pulse tick to vay

### palette
    PAL+   ( n -- )   -- advance palette offset by n
    PAL@   ( -- n )   -- push current palette offset

### fingerprint
    FP     ( -- fp )  -- push current fingerprint
    FP!    ( fp -- )  -- set device state from fingerprint string (lookup mode)
    FP:Z   ( fp z -- key ) -- compose message key

### station
    .TAI  .VAY  .GRI  .KAA  .AAV  .HEH  .POH  .SOL
           -- push the coord of the named station

### output
    EMIT   ( key value -- )  -- publish message to vay
    .S     ( -- )            -- print stack (debug)
    .FP    ( -- )            -- print fingerprint + rot + pal state

### control
    : name ... ;   -- define word
    IF ELSE THEN
    BEGIN UNTIL
    DO LOOP

---

## example programs

    -- spin the device three steps and report fingerprint
    3 ROT+ .FP

    -- jump to VAY station, advance palette by 7, emit state
    .VAY ORD 1 - 36 MOD  ( map_rot to land VAY at top )
    ROT+ 7 PAL+ FP .FP EMIT

    -- pulse at 0.15Hz for one revolution (36 ticks)
    36 0 DO TICK 6670 SLEEP LOOP

    -- define a season-seek word
    : SPRING  89 PAL! 18 ROT+ .FP ;
    SPRING

---

## gesture mapping (future)

The gesture decoder (touchscreen/pen/mouse) produces Forth words:
- slow drag right  →  ROT-
- fast fling left  →  n ROT+  (n from velocity)
- tap station hex  →  .VAY FP! (or whichever station)
- hold + drag      →  scrub: continuous ROT+ at drag rate

The gesture layer is a Forth word generator. vay sees only the resulting words.
The interpreter is the translation layer between physical input and bus events.

---

## open questions

- SLEEP granularity: nanosleep or vay heartbeat signal?
- Persistent vocabulary: save defined words to disk between sessions?
- Multi-client: two Forth sessions sending to same vay topic — last-writer-wins or merge?
- AI interface: natural language → Forth transpilation (the original motivation)
