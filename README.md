# vay

A message bus. Exactly-once delivery. Durable log. No suit.

VAY is the channel station in the KaamOS hex grid — the place where things in transit live.
This is the software that lives there.

Built as a lightweight alternative to Kafka for event-driven desktop and audio RT measurement
workflows. Go. Binary framing. Write-ahead log. Fingerprint keys.

Part of the same universe as [kaiku](https://github.com/Jakko-KAAMOS/kaiku).

A side hustle.

---

## what it is

- ordered, durable message log
- exactly-once producer semantics via ack-on-fsync
- consumer offset tracking
- fingerprint key as first-class type (hex grid state address)
- TCP transport, binary protocol

## what it is not

- Kafka
- a queue
- finished

---

## the station

Messages enter the void and exit north.
