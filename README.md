# redis-nv

Redis is an in-memory data store used as a cache, a message broker and
a database. Clients talk to it over **RESP**, the REdis Serialization
Protocol, specified in Redis's own
[protocol specification](https://redis.io/docs/latest/develop/reference/protocol-spec/).
This package speaks RESP2 and RESP3 in novo-lang: a codec that
performs nothing, and a client over `std.net` built on it. It covers
the commands, pub/sub, transactions, Lua scripts and cluster routing.

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What the protocol is

Everything on the wire is a **frame**: one type byte, then the frame's
content, then a carriage return and a line feed. A client sends a
**command** as an array of byte strings, and the server answers one
frame.

**RESP2** is the original protocol and has five types. **RESP3** adds
seven more, and a server speaks RESP2 until the client sends
`HELLO 3`.

| Byte | Frame | Protocol |
| --- | --- | --- |
| `+` | A short unstructured string, such as `OK` or `PONG` | both |
| `-` | An error | both |
| `:` | A 64-bit signed integer | both |
| `$` | A byte string with its length in front | both |
| `*` | An array of frames | both |
| `_` | The absent value; RESP2 spells it `$-1` or `*-1` | RESP3 |
| `,` | A double, which carries `inf`, `-inf` and `nan` | RESP3 |
| `#` | A boolean, `t` or `f` | RESP3 |
| `=` | A string with a three-character format hint | RESP3 |
| `(` | An integer too large for 64 bits, as decimal text | RESP3 |
| `%` | A map, as alternating keys and values | RESP3 |
| `~` | A set, whose order means nothing | RESP3 |
| `>` | A **push** | RESP3 |

**A push is not a reply.** It is a message the server sends because
something happened, not because the client asked. Pub/sub messages,
client-side caching invalidations and `MONITOR` output all arrive that
way. A client that took a push for the reply to a pending command
would answer that command with somebody else's message, and every
later reply would be off by one, from that moment on.

In RESP2 there is no push type. A pub/sub message arrives as an
ordinary array, which is indistinguishable from an array reply. That
ambiguity cannot be resolved, so Redis forbids a subscribed RESP2
connection from running any other command.

Some error replies are **instructions rather than failures**. In a
cluster, `MOVED` says the key's slot lives on another node and `ASK`
says the slot is migrating. `NOSCRIPT` says a script's digest is not
cached here. `LOADING` says the server is still reading its dataset
from disk. Each of those is a routine step in a working client.

A **cluster** divides the keyspace into 16,384 **slots**. A key's slot
is the CRC-16/XMODEM of its **hash tag** modulo 16,384. The hash tag
is the text between the first `{` and the first `}` after it, when
that text is not empty, and the whole key otherwise. It is what lets a
caller force two keys onto one node.

| Quantity | Value |
| --- | --- |
| Service port | 6379 |
| Cluster slots | 16,384 |
| CRC-16/XMODEM polynomial | 0x1021 |
| CRC-16/XMODEM initial value | 0x0000, with no reflection and no final xor |
| Its check value over `123456789` | 0x31C3 |
| CRC-16/CCITT-FALSE's check value over the same, for comparison | 0x29B1 |
| Largest value Redis stores | 512 MB, so a frame's ceiling is the caller's |

## Install

```
novo pkg add redis-nv
```

## Example

```novo
use std.bytes
use rdclient
use rdcmd
use rderror
use rdresp

fn main() [io, net, time]
    // Where to connect. `plain_transport` is the standard library's
    // socket; a program that wants TLS supplies its own.
    match rdclient.connect(rdclient.default_options(), rdclient.plain_transport())
        Err(f) => println(rderror.describe(f))
        Ok(c)  =>
            // SET session:7 to a value that expires in thirty seconds.
            let cmd = rdcmd.set(bytes.from_str("session:7"), bytes.from_str("ada"),
                                RdSeconds(30), RdSetAlways, false)
            match rdclient.call(c, cmd)
                Err(f) => println(rderror.describe(f))
                Ok(w)  =>
                    // Read it back. A missing key is `None`, not an error.
                    match rdclient.call(w.conn, rdcmd.get(bytes.from_str("session:7")))
                        Err(f) => println(rderror.describe(f))
                        Ok(r)  =>
                            match rdresp.as_bulk(r.reply)
                                Err(f)      => println(rderror.describe(f))
                                Ok(None)    => println("no such key")
                                Ok(Some(v)) => println(bytes.to_str(v))
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: redis-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `rdresp` | Every frame type in both protocols, the framing, the encoder and decoder, and the readers that turn a frame into a value. |
| `rdcmd` | A command as a value, its encoding, and typed constructors for the strings, hashes, lists, sets, sorted sets and streams. |
| `rdclient` | The socket, the `HELLO` handshake, one call, a pipeline, a blocking call, and the queue of pushes that arrived meanwhile. |
| `rdpubsub` | A subscription as a value the caller pumps, the four subscribe commands, and the reading of a push as a message. |
| `rdtx` | `WATCH`, `MULTI`, queueing, `EXEC` and `DISCARD`, and the three things an `EXEC` can mean. |
| `rdscript` | Lua scripts by body or by digest, the digest computed locally, the client's cache of known digests, and the function commands. |
| `rdcluster` | The slot arithmetic, the hash tag, the CRC, the slot map, and the two redirections with their different rules. |
| `rderror` | Every reason a call did not answer, the split between an error and an instruction, and the reading of a server's error reply. |

`rdresp`, `rdcmd`, `rdcluster`'s slot arithmetic and `rderror` perform
no input or output. That is what lets the protocol be tested against
captures with no server running, and it is why the cluster slot, the
number everything routes on, has a unit test rather than an
integration test. `rdclient`, `rdpubsub`, `rdtx` and `rdscript`
declare `[net]`, and `[time]` where a deadline is consulted, which for
Redis means every blocking command. No module here declares `[io]`.

`rdresp` encodes and decodes in both directions, because a frame is a
frame whichever end sent it. A RESP server would take the module
unchanged.

## How to choose an entry point

**`rdclient.call` sends one command and waits for its reply.** It
routes pushes out of the way before it looks for one.

**`rdclient.pipeline` sends several commands in one write.** Redis
answers them in order, and the round trip is paid once.

**`rdclient.blocking_call` is for a command that waits on the
server**, such as `BLPOP`. It takes the command's own timeout into
account, so the socket deadline is not set below it.

**`rdclient.send` and `read_reply` are the two halves**, for a caller
that wants to interleave.

**`rdcmd.command` and `command_text` build a command this package does
not name.** Give the arguments, the positions of the keys and the
reply shape expected.

**`rdpubsub.subscription` is a value the caller pumps.**
`rdpubsub.pump` advances it one message at a time and answers.

**`rdresp` and `rdcmd` together are the whole protocol with no
socket.** That is the path for a proxy, a capture reader and a test.

## The rules a user needs

1. **A push is not a reply, and `rdresp.is_push` is the predicate.**
   `rdclient.call` routes on it before it looks for a reply, and
   `rdclient.take_pushes` hands over the ones that arrived.
2. **A RESP2 connection that is subscribed may run no other command.**
   `rdpubsub.requires_second_connection` answers whether this
   connection needs a second one, and
   `rdpubsub.allowed_while_subscribed` answers whether a given command
   may be sent.
3. **One socket read is not one frame.** `rdresp.frame_length` says
   how much of a buffer is one frame, and the client reassembles
   across reads.
4. **Bound the frame size before the first read.**
   `rdresp.frame_length` takes `max_bytes`. The ceiling is the
   caller's, because a Redis value can legitimately be 512 MB and the
   length field is a number the wire supplies.
5. **A missing key is `None`, not an error.** It is the commonest
   thing that happens to a Redis client. `rdresp.as_bulk` answers
   `Ok(None)`.
6. **`MOVED` and `ASK` are instructions, not failures.**
   `rderror.is_instruction` is the predicate. A client that surfaced
   `MOVED` to its caller turned a routine redirect into an application
   error.
7. **`MOVED` updates the slot map and `ASK` does not.**
   `rdcluster.updates_map` says which. Treating an `ASK` as a `MOVED`
   points the map at the wrong node for the whole of a migration.
8. **An `ASK` redirect needs an `ASKING` command first**, on the
   connection to the new node, before the retried command.
   `rdcluster.needs_asking` says so. Without it the new node answers
   `MOVED` straight back, and the client loops.
   `rdcluster.redirect_limit` bounds how many redirections one command
   may follow.
9. **The cluster CRC is XMODEM, not CCITT-FALSE.** They share the
   polynomial 0x1021 and differ in the initial value: 0x0000 against
   0xFFFF. A client using the wrong one computes a different slot for
   every key, sends every command to the wrong node, is redirected,
   and works anyway. Correct results, twice the latency, and nothing
   in any log. `rdcluster.crc16_xmodem` is the one this package uses.
10. **The hash-tag rule is exact.** It is the first `{`, the first `}`
    after it, and the content between them must be non-empty. So
    `{a}b{c}` hashes `a`, `foo{}bar` hashes the whole key, and
    `foo{}{bar}` hashes `bar`. `rdcluster.hash_tag` is the rule, and
    `rdcluster.same_slot` says whether a multi-key command's keys are
    all on one node.
11. **A plain `SET` clears any expiry the key had.** `RdKeepTtl` is
    the option a caller updating a cached value almost always wants
    and almost never writes. `RdExpiry` names all six possibilities,
    and `rdcmd.set` puts them in the order the server expects.
12. **A subscribe confirmation arrives as a push too.**
    `rdpubsub.of_push` tells a confirmation from a message, and
    `rdpubsub.is_message` is the check. A client that reads every push
    as a message reports a phantom message per subscribe, forever.
13. **`MULTI` has no rollback.** A command that fails at run time
    inside `MULTI`/`EXEC` does not stop the others, and `EXEC` still
    succeeds. The failure is one element of the reply array.
    `rdtx.reply_failed` is the check, and `rdtx.failed_positions` says
    which elements failed.
14. **A null `EXEC` reply means a watched key changed.** Nothing ran,
    and the caller retries the whole read-modify-write.
    `RdExecOutcome` has three variants so that "it ran", "retry" and
    "a queued command was malformed" cannot be collapsed.
15. **`EXEC` and `DISCARD` clear every `WATCH`**, whether or not they
    succeeded. A second attempt that does not watch again watches
    nothing, and its `EXEC` succeeds against data that changed
    underneath it.
16. **Send `EVALSHA` first and fall back to `EVAL` on `NOSCRIPT`.**
    `rdscript.digest_of` computes the digest locally, so the
    optimistic call costs no round trip in the steady state.
    `RdScriptCache` remembers which digests this client has seen.
17. **Set the socket deadline above a blocking command's own
    timeout.** A `BLPOP` with a thirty-second timeout on a socket with
    a five-second deadline disconnects the client from itself.
    `rdclient.blocking_call` is the call that gets it right.
18. **A `HELLO` that answers an error is a RESP2 server, not a
    failure.** Servers older than Redis 6 do not have the command.
    `rdclient.connect` treats the error as the discovery it is, and
    `rdclient.protocol_of` says which protocol the connection ended up
    speaking.

## What is not included

- **Sentinel.** Finding a master through a sentinel quorum is a
  different topology from cluster routing, and it wants its own
  module.
- **A connection pool.** `RdConn` is a value and a program holds as
  many as it wants. A Redis connection is cheap, and the interesting
  decision is pipelining depth rather than connection count.
- **Client-side caching.** RESP3's tracking invalidations arrive as
  pushes and are carried through, so a caller can build one. A cache
  with an invalidation protocol is a package rather than a function.
- **`KEYS`.** It blocks the server for the length of the scan, and on
  a production instance that is an outage. `rdcmd.scan` is the
  cursor-based alternative. A caller that really wants `KEYS` builds
  it with `rdcmd.command`, having typed the name.
- **A callback interface for subscriptions.** A callback would put the
  caller's code inside this package's read loop, and therefore inside
  this package's effect row: a handler that logged would make the pump
  declare `[io]`, and every consumer would pay for the widest handler
  any consumer wrote. `rdpubsub.pump` costs the caller exactly what
  the caller does.
- **A TLS implementation.** `RdTransport` is the seam, and
  `rdclient.plain_transport` is the `std.net` pair.
- **Asynchronous calls.** Every call blocks its task. `std.net` has
  `recv_async` and this package does not use it yet.
- **A big-integer parser.** `RdBigNumber` carries the decimal text. A
  caller that needs the value takes it to `std.bigint`, and one that
  only logs it does not pay for that.

## Related packages

- [crypto-nv](https://novo-lang.org/packages/crypto-nv) is the SHA-1
  behind `rdscript.digest_of`. SHA-1 is Redis's choice here and is not
  a security claim: the digest identifies a script in a cache this
  client filled.
- [crc-nv](https://novo-lang.org/packages/crc-nv) is not a dependency.
  It publishes CRC-16/CCITT-FALSE and CRC-16/MODBUS, and has no public
  constructor taking a parameter set, so XMODEM cannot be built from
  its surface. `rdcluster.crc16_xmodem` carries its own table. A
  parameterised constructor on crc-nv is what would let this package
  take the dependency and delete the table.
- [postgres-nv](https://novo-lang.org/packages/postgres-nv) and
  [mysql-nv](https://novo-lang.org/packages/mysql-nv) are the same
  shape for a SQL server: a codec half that performs nothing and a
  client half over `std.net`.
- `std.collections` in the standard library has `HashMap`, which is a
  key-value store in this process. Take Redis when the store has to
  outlive the process or be shared between several.
- `std.net` in the standard library is where the bytes come from and
  where they go.

## Tests

```bash
novo test tests/rdresp_tests.nv      # 10 tests: the frames, both protocols
novo test tests/rdcluster_tests.nv   #  8 tests: the slot, the tag, the redirections
novo test tests/rdhost_tests.nv      # 10 tests: the client, pub/sub and transactions
```

The wire is Redis's own protocol specification, and the cluster
specification is where the slot arithmetic and the two redirections
come from. `redis-rs` and `redis-py` are the reference
implementations for the shape of the API.

No test opens a socket. The suite checks the CRC against its published
check value, `0x31C3` over `123456789`, and against CCITT-FALSE's
`0x29B1` so the two cannot be confused. It checks the three hash-tag
corners named in rule 10, that a push is not paired with a pending
command, that all three spellings of null decode to one value, that a
subscribe confirmation is not a message, that a null `EXEC` reply is
read as contention rather than as an error, and that an error inside a
successful `EXEC` is reported.

The tests compile today and fail at run, each on the
`not implemented: redis-nv.<module>.<fn>` panic that is its body. That
is the expected state of an interface release. They turn green one at
a time as bodies land.

## Implementation status

Nothing is implemented. Every function below is a `todo()`.

| Module | Public surface |
| --- | --- |
| `rdresp` | `frame_length`, `decode`, `encode`, `encode_null`, `is_push`, `is_error`, `is_resp3_only`, `type_byte`, `type_name`, `as_bulk`, `as_text`, `as_int`, `as_double`, `as_bool`, `as_list`, `as_pairs`, `as_fault` |
| `rdcmd` | `command`, `command_text`, `encode`, `encode_pipeline`, `keys_of`, `name_of`; `hello`, `auth`, `ping`, `select_db`, `info`; `get`, `set`, `getdel`, `getex`, `del`, `exists`, `incrby`, `incrbyfloat`, `mget`, `mset`, `expire`, `ttl_ms`; `hget`, `hset`, `hdel`, `hgetall`, `hincrby`; `push`, `pop`, `blocking_pop`, `lrange`, `llen`; `sadd`, `srem`, `smembers`, `sismember`; `zadd`, `zscore`, `zrange`, `zrange_by_score`, `zrem`; `xadd`, `xrange`, `xread`, `xgroup_create`, `xreadgroup`, `xack`; `scan`, `scan_key` |
| `rdclient` | `plain_transport`, `default_options`, `connect`, `close`, `protocol_of`, `call`, `pipeline`, `blocking_call`, `send`, `read_reply`, `take_pushes` |
| `rdpubsub` | `requires_second_connection`, `allowed_while_subscribed`, `subscription`, `subscribe`, `psubscribe`, `ssubscribe`, `unsubscribe`, `punsubscribe`, `pump`, `drain`, `of_push`, `is_message`, `publish`, `spublish` |
| `rdtx` | `transaction`, `watch`, `unwatch`, `watching`, `multi`, `queue`, `will_abort`, `exec`, `discard`, `reply_failed`, `failed_positions` |
| `rdscript` | `digest_of`, `script`, `script_by_digest`, `cache`, `is_known`, `remember`, `forget`, `invalidate`, `evalsha`, `eval`, `eval_readonly`, `script_load`, `script_exists`, `script_flush`, `fcall`, `fcall_readonly`, `function_load` |
| `rdcluster` | `slot_count`, `key_slot`, `hash_tag`, `crc16_xmodem`, `same_slot`, `command_slot`, `empty_map`, `cluster_shards`, `cluster_slots`, `asking`, `readonly`, `map_of_reply`, `owner_of`, `replicas_of`, `redirect_of`, `needs_asking`, `updates_map`, `with_moved`, `redirect_limit` |
| `rderror` | `describe`, `of_error_reply`, `error_code`, `is_instruction`, `is_fatal`, `is_retryable` |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
