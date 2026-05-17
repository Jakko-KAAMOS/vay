# vay — behaviour specification v0.1

## subscriber modes

vay drives `kaamos_ordinal` (the antikythera screensaver) in one of two modes.
The subscriber does not select the mode; the message shape determines it.

### pulse mode

vay emits a tick message every 6.67s.
Subscriber response: increment `map_rot` by 1, increment `pal_offset` by 1.
State is implicit — it accumulates in arrival order.
The sequence is the log.

### lookup mode

vay emits a message with a fingerprint key: `{[...]}}:z`
Subscriber response: decode `map_rot` and `pal_offset` from the key, set state directly.
Enables: log replay, seek to named position, jump to station address.
The screensaver becomes seekable.

## message shape

    pulse:  { type: TICK }
    lookup: { type: STATE, key: "{[1,36],[2,0,5],[3,4]}:0" }

If `key` is absent or null → pulse semantics.
If `key` is present → lookup semantics.

The HUD fingerprint on screen confirms arrival and decoded state.

## subscriber behaviour — both modes

On message receipt:
1. Parse key if present → extract `map_rot`, `pal_offset`; else increment both
2. Update visual state
3. Render frame (60Hz GTK3/Cairo loop continues independently)
4. Emit ack

The render loop is not blocked by vay. Frame rate is decoupled from event rate.
State updates are applied between frames.

## v0.1 scope

- TCP socket listener replaces `GLib.timeout_add` as state source
- Pulse mode only in v0.1; lookup mode in v0.2
- No consumer group tracking in screensaver — single subscriber, no offset management
- vay broker not yet implemented; v0.1 screensaver accepts raw TCP line `TICK\n`
  or `STATE {key}\n` for manual testing

## open

- reconnect behaviour on vay disconnect (fall back to clock tick? halt?)
- backpressure: screensaver slow to ack — vay drops or queues?
- multi-subscriber: two screens, one vay topic
