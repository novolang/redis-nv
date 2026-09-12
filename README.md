# redis-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

A Redis client in novo-lang: RESP2 and RESP3 as a codec that performs
nothing, and a client over `std.net` built on it.

RESP is a small protocol — twelve type bytes and a length prefix — and
the reason a client is more than an afternoon's work is not the parsing.
It is the half-dozen rules that are invisible until they are wrong: a
push that is not a reply, a `MOVED` that is not an error, a `SET` whose
expiry option differs from another by one letter, a cluster slot whose
CRC has a near-identical twin.  Each of those is a public function here,
because each of them is a silent bug elsewhere.

Eight modules, and a reader should know which one they are on.

| surface | module | reach for it when |
| --- | --- | --- |
| the **frames** | `rdresp` | anything. Start here |
| the **commands** | `rdcmd` | you are building something to send |
| the **connection** | `rdclient` | you want to send it |
| the **subscriptions** | `rdpubsub` | you are listening |
| the **transactions** | `rdtx` | you are using `MULTI`/`WATCH` |
| the **scripts** | `rdscript` | you are running Lua |
| the **cluster** | `rdcluster` | there is more than one node |
| the **faults** | `rderror` | something came back wrong |

## Adding it, and checking it

```bash
novo pkg add redis-nv             # into your novo.toml
novo pkg build                    # type- and effect-check the package
novo test --isolate tests/rdresp_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion fails with `not implemented: redis-nv.<module>.<fn>`.  They
turn green one at a time as bodies land.

## The one example that will work

```novo
use rdclient
use rdcmd
use rdresp
use std.bytes

// A value, or nothing, with a thirty-second life.
fn cache_get(key: Str) -> ?Bytes [net, time]
    match rdclient.connect(rdclient.default_options(), rdclient.plain_transport())
        Err(f) => None
        Ok(c) =>
            match rdclient.call(c, rdcmd.get(bytes.from_str(key)))
                Err(f) => None
                Ok(r)  =>
                    match rdresp.as_bulk(r.reply)
                        Ok(v)  => v
                        Err(_) => None
```

A missing key is `None` and not an error — that is the commonest thing
that happens to a Redis client and it is not a failure.

## The layer, and why

`host`, and half the package is `[]`.

`rdresp`, `rdcmd`, `rdcluster`'s slot arithmetic and `rderror` perform
nothing: bytes in, frames out; frames in, bytes out; a key in, a slot
out.  That is what lets the protocol be tested against captures with no
server running, and it is why the cluster slot — the number everything
routes on — has a unit test rather than an integration test.

`rdclient`, `rdpubsub`, `rdtx` and `rdscript` connect.  They declare
`[net]`, and `[time]` where a deadline is consulted, which for Redis
means every blocking command.  `[io]` is not owed: `std.net`'s free
functions declare `[net]` alone, and no module here prints.

**The `core` half is a MODULE split, and unlike postgres-nv this one
has a second caller in sight.**  `rdresp` decodes and encodes in both
directions already — a frame is a frame whichever end sent it — so a
RESP SERVER would take the same module unchanged.  If one is written,
`rdresp` moves out as `resp-nv` with no signature change and both ends
take it.  Named here so that is a decision waiting rather than an
omission.

## The load-bearing interface

`rdresp.RdFrame`, and the predicate that comes with it:
`rdresp.is_push`.

```novo norun:pseudo
pub enum RdFrame
    RdSimple(v: Str)      // +      RdError(v: Str)          // - !
    RdInt(v: Int)         // :      RdBulk(v: Bytes)         // $
    RdArray(v: [RdFrame]) // *      RdNull                   // _ $-1 *-1
    RdDouble(v: Float)    // ,      RdBool(v: Bool)          // #
    RdVerbatim(…)         // =      RdBigNumber(v: Str)      // (
    RdMap(…)              // %      RdSet(v: [RdFrame])      // ~
    RdPush(v: [RdFrame])  // >
```

**Why `is_push` is the predicate the whole client turns on.**  In RESP2
a pub/sub message arrives as an ordinary ARRAY — `["message", channel,
payload]` — which is indistinguishable from an array REPLY to a command.
A client cannot tell one from the other, which is exactly why Redis
FORBIDS a subscribed RESP2 connection from running any other command:
the ambiguity is unresolvable, so the protocol removes the case.

RESP3 fixes it with a distinct type.  A `>` frame is not a reply to
anything, which is what lets one connection carry commands and
subscriptions at once — and it is what client-side caching's
invalidation messages and `MONITOR` also ride on.  A client that decoded
`>` and paired it with the next pending command answered that command
with somebody else's message AND left every later reply off by one: a
failure that starts one subscription after the bug and never stops.

So `is_push` is public, `rdclient.call` routes on it before it looks for
a reply, and `rdpubsub.requires_second_connection` answers the question
that falls out of it.

**The two protocols are one decoder, and that is not a convenience.**  A
server speaks RESP2 until a client sends `HELLO 3`, so a decoder that
only knew RESP3 could not read the reply to the `HELLO` that asks for
it.  A server older than Redis 6 answers an error to `HELLO`, which is
not a failure — it is the discovery that this is a RESP2 server — and
`rdclient.connect` treats it that way.

## What a client gets wrong quietly, and where each one has a name

| the mistake | what it costs | where it is named |
| --- | --- | --- |
| a push paired with a pending command | every later reply off by one | `rdresp.is_push` |
| one socket read treated as one frame | works locally, fails pipelined | `rdresp.frame_length` |
| the length field trusted | an allocation the wire asked for | `frame_length`'s `max_bytes` |
| `MOVED` surfaced as an error | a routine redirect becomes an app failure | `rderror.is_instruction` |
| `ASK` treated as `MOVED` | the slot map points at the wrong node for the whole migration | `rdcluster.updates_map` |
| `ASKING` forgotten before an `ASK` redirect | a `MOVED` straight back, and a loop | `rdcluster.needs_asking` |
| CRC-16/CCITT-FALSE instead of XMODEM | correct results at twice the latency, silently | `rdcluster.crc16_xmodem` |
| a hash tag read loosely | `{a}b{c}` and `foo{}{bar}` routed wrong | `rdcluster.hash_tag` |
| a subscribe confirmation read as a message | a phantom message per subscribe, forever | `rdpubsub.of_push` |
| a null `EXEC` reply read as an error | normal contention becomes an app error | `rdtx.RdWatchBroken` |
| an error inside a successful `EXEC` ignored | there is no rollback, so it happened | `rdtx.reply_failed` |
| a socket deadline below a `BLPOP`'s | the client disconnects itself | `rdclient.blocking_call` |

## The cluster slot, and the CRC that is not the other one

`rdcluster.key_slot` is `CRC16-XMODEM(hash_tag(key)) mod 16384`.

XMODEM is polynomial 0x1021, **initial value 0x0000**, no reflection, no
final xor.  CCITT-FALSE is the same polynomial with an initial value of
**0xffff**.  A client that used the second computes a different slot for
every key, sends every command to the wrong node, gets a `MOVED`, and
**works anyway** after one extra round trip.  Correct results, twice the
latency, and nothing in any log — which is why the check value is
asserted directly: XMODEM over `"123456789"` is `0x31C3`, CCITT-FALSE is
`0x29B1`, and the empty input is `0x0000` against `0xFFFF`.

**This is also why crc-nv is not a dependency.**  It publishes
CCITT-FALSE and MODBUS, and has no public constructor taking a parameter
set, so XMODEM cannot be built from its surface.  The table is here, and
**the row this asks for is a `crc.custom(poly, init, xor_out, width,
reflected)` on crc-nv** — after which this package takes the dependency
and deletes its table.  That is the missing row this lane found.

The hash-tag rule is exact and narrow, and each edge below is a shape a
loose implementation gets wrong while mostly working: the **first** `{`,
the **first** `}` after it, and the content between them must be
**non-empty**.  So `{a}b{c}` hashes `a`, `foo{}bar` hashes the whole key,
and `foo{}{bar}` hashes `bar`.

## The subscription is a value, not a callback

`rdpubsub.RdSubscription` holds what is subscribed and what has arrived;
`pump` advances it one message at a time and answers.

A callback would put the caller's code inside this package's read loop,
which means inside this package's effect row: a handler that logged
would make the pump `[io]`, one that wrote a file `[fs]`, and every
consumer would pay for the widest handler any consumer wrote.  A value
the caller pumps costs the caller exactly what the caller does.

## What `MULTI` is not

Two properties a caller will assume and should not:

- **There is no rollback.**  A command that fails at runtime inside
  `MULTI`/`EXEC` does not stop the others, and `EXEC` still succeeds;
  the failure is one element of the reply array.  `rdtx.reply_failed` is
  the check a caller would not think to make, because `EXEC` succeeded.
- **A null `EXEC` reply is the contention signal, not an error.**  It
  means a `WATCH`ed key changed and nothing ran, and the caller retries
  the whole read-modify-write.  `RdExecOutcome` has three variants so
  that "it ran", "retry" and "a command was malformed" cannot be
  collapsed.

And one rule a retry loop gets wrong: **every `WATCH` is cleared by
`EXEC` or `DISCARD`**, succeeded or not.  A second attempt that did not
re-`WATCH` watches nothing, and its `EXEC` succeeds against data that
changed underneath it.

## novokv, and the subset it does not implement

The plan's row says redis-nv is "also novokv's client".  **It cannot be,
as things stand, and that is a finding rather than a gap here.**

`orbit/novokv` speaks a line-oriented text protocol in the memcached
tradition — `GET`, `SET <key> <ttl_ms> <nbytes>` plus a body, `DEL`,
`INCR`, `STATS`, `QUIT` — with `VALUE <n>` and `STORED` replies.  There
is no RESP anywhere in it: not a type byte, not a bulk string, not an
array.  A RESP client cannot talk to it, and a package named after RESP
should not grow a second wire protocol to try.

**The subset is exact, though, which is what makes the fix cheap.**
Every one of novokv's six verbs has a RESP command with the same
meaning, and this package already declares all six:

| novokv | redis-nv | note |
| --- | --- | --- |
| `GET k` | `rdcmd.get` | `NOT_FOUND` is a null bulk string |
| `SET k ttl n` + body | `rdcmd.set` with `RdMilliseconds` | novokv's ttl is already milliseconds, which is `PX` |
| `DEL k` | `rdcmd.del` | the reply is a count rather than `DELETED` |
| `INCR k delta` | `rdcmd.incrby` | novokv's delta may be negative, like `INCRBY`'s |
| `STATS` | `rdcmd.info` | novokv answers per shard; `INFO` is per server |
| `QUIT` | — | one command, and closing the socket does as well |

So the row that closes it is **a RESP front door on novokv**: a listener
that accepts inline arrays of bulk strings and answers `+`, `:`, `$` and
`*`.  That is `rdresp` used in the other direction, which is the second
caller the module-split section names — and after it, this package is
novokv's client with no change at all.  Until then, the README says so
rather than the plan's row implying otherwise.

## One dependency

**crypto-nv**, for SHA-1, for `EVALSHA` and nothing else.

A client can get a script's digest from the server with `SCRIPT LOAD` —
a round trip, on a connection, before the script can be used.  Computing
it locally is what lets a client send `EVALSHA` optimistically and fall
back to `EVAL` on `NOSCRIPT`, which costs zero round trips in the steady
state and is the pattern every Redis client settled on.  `digest_of` is
public because a caller logging which script ran needs the same forty
characters.

SHA-1 here is Redis's choice and is not a security claim: the digest
identifies a script in a cache the client itself populated.

## What is out of scope, out loud

**Sentinel.**  Discovering a master through a sentinel quorum is a
different topology from cluster routing and wants its own module or its
own row.

**A connection pool.**  `rdclient.RdConn` is a value and a program holds
as many as it wants; the pooling questions Redis has are not
PostgreSQL's — a connection is cheap, and the interesting decision is
pipelining depth rather than connection count.

**Client-side caching.**  RESP3's tracking invalidations arrive as
pushes and `RdOtherPush` carries them, so a caller can implement it; a
cache with an invalidation protocol is a package rather than a function.

**`KEYS`.**  Deliberately not in `rdcmd`.  It blocks the server for the
length of the scan, and on a production instance that is an outage.  A
caller that really wants it builds it with `rdcmd.command`, having typed
the name.

**`async`.**  Every call blocks its task.  `std.net` has `recv_async`
and this package does not use it yet; the row that wants it is a program
holding many subscriptions per cell.

## The reference implementations

redis-rs and redis-py, for the API shape; Redis's own
[protocol specification](https://redis.io/docs/latest/develop/reference/protocol-spec/)
for RESP2 and RESP3, which is what the module headers transcribe; the
cluster specification for the slot arithmetic and the two redirections.

The implementation lane's gate is a real `redis-server`: one node for
the commands, a three-node cluster for the routing, and `redis-cli` for
comparison.

## Status

Interface only.  Eight modules, 145 public functions, every body a
`todo()`.

- `novo pkg build` — clean, 8 modules checked.
- `novo test` — three suites, all red, every failure `not implemented`.
- `scripts/shard_audit.sh --strict` — `effect-budget`, `dep-layer`,
  `no-discharge-in-core`, `doc-examples` and `docs-pub` green; `test`
  red by design.
