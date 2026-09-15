# Changelog

All notable changes to redis-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.2 — 2026-09-15

- README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-11

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `rdresp` — all twelve RESP type bytes in one decoder, the framing
  check against the caller's ceiling, `is_push`, and the readers that
  hide the two places RESP2 and RESP3 differ from a caller.
- `rdcmd` — a command as a list of byte strings, typed builders for
  the string, hash, list, set, sorted-set and stream families, and a
  first-class `command` escape hatch that declares its own key
  positions.
- `rdclient` — the socket, the `HELLO` negotiation that can fail
  benignly, the read loop that routes a push before it looks for a
  reply, pipelining, and `blocking_call`'s deadline arithmetic.
- `rdpubsub` — a subscription as a value the caller pumps, all four
  message kinds, and the question of whether this connection needs a
  second one.
- `rdtx` — `MULTI`/`WATCH`/`EXEC` with three outcomes, the queueing
  replies read as they go out, and the check for an error inside a
  successful `EXEC`.
- `rdscript` — the optimistic `EVALSHA`-then-`EVAL` pattern, the
  digest computed locally, a per-connection cache with a fallback
  counter, and `FCALL`.
- `rdcluster` — CRC-16/XMODEM, the hash-tag rule at its edges, the
  slot map as a value, and the two redirections kept apart.
- `rderror` — the code word split from the prose, and the four replies
  that are instructions rather than failures.

### Known

- **The load-bearing interface is `rdresp.RdFrame` and
  `rdresp.is_push`.**  A RESP2 pub/sub message is an ordinary array
  and cannot be told from a reply, which is why Redis forbids other
  commands on a subscribed RESP2 connection; RESP3's push type is what
  lifts that, and a client that paired a push with a pending command
  leaves every later reply off by one.
- **The cluster slot is CRC-16/XMODEM, not CCITT-FALSE** — the same
  polynomial with a different initial value.  Getting it wrong
  produces correct results at twice the latency with nothing in any
  log, so the check value is asserted directly.
- **crc-nv is not a dependency and the reason is a gap in crc-nv**: it
  has no public constructor taking a parameter set, so XMODEM cannot
  be built from its surface.  The row this asks for is
  `crc.custom(poly, init, xor_out, width, reflected)`.
- **`MOVED` and `ASK` are different instructions**, and only one of
  them updates the slot map.
- **`MULTI` has no rollback and a null `EXEC` is the contention
  signal**, which is why the outcome has three variants.
- **The subscription is a value the caller pumps**, so a handler's
  effects are the caller's rather than this package's.
- **novokv is NOT reachable from this client**: it speaks a
  memcached-tradition text protocol, not RESP.  The README names the
  exact six-verb subset and the row that would close it — a RESP front
  door on novokv, which is `rdresp` used in the other direction.
- **One dependency**, crypto-nv, for SHA-1 and `EVALSHA` alone.
- **Out of scope and said so**: Sentinel, a connection pool,
  client-side caching, `KEYS`, and `async` entry points.
