# RFC: Valkey GLIDE for Rust without redis-rs

Status: Draft · Target: `valkey-glide` 0.11 (breaking batch) → 1.0 (GA) · Tracking epic: [#6915](https://github.com/valkey-io/valkey-glide/issues/6915)

## Abstract

Rust projects that talk to Valkey today almost always depend on
[redis-rs](https://github.com/redis-rs/redis-rs) (the `redis` crate), whose
command surface follows Redis rather than Valkey. GLIDE already ships a native
Rust client (`valkey-glide` 0.10.0, lib `glide`) built directly on `glide-core`.
This RFC defines what "no redis-rs needed" means, records which parts are
already true, and specifies the remaining work to reach a GA client that a Rust
project can adopt without the `redis` crate, its types, or its ecosystem
adapters.

## Current state (verified on `feature/rust` at `66a448cdd`)

The RFC builds on existing code. It does not propose a new crate.

| Area             | State                                                                                                                                                                                                                                                                                                                 |
| ---------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Crate            | `rust/`, package `valkey-glide` 0.10.0, published to crates.io (2,345 downloads). Edition 2024, MSRV 1.94.1, `#![forbid(unsafe_code)]`, `#![deny(missing_docs)]`.                                                                                                                                                     |
| Engine           | Links `glide-core` 0.3.0 directly (no FFI, no socket layer). `glide-core` uses `glide-core-engine` 0.2.0, the GLIDE-owned fork of redis-rs **0.25.2** (BSD-3-Clause), whose `[lib] name` is still `redis`.                                                                                                            |
| Public API       | GLIDE-owned types only: `ValkeyValue`, `GlideError`, `Cmd`, `Pipeline`, `Route`, `ToValkeyArgs`, `FromValkeyValue`. Every use of fork types in `rust/src` is `pub(crate)` (`Cmd::into_redis`, `ValkeyValue::from_redis`, `GlideError::from_redis_error`, …).                                                          |
| Command surface  | `AsyncCommands`/`Commands` plus typed variants, generated from a hand-maintained table that is source-compatible with redis-rs **1.7.0** and checked by a fail-closed parity guard (`src/parity_tests/`). GLIDE extension traits cover streams, geo search, JSON, Pub/Sub, scripting/functions and server management. |
| Clients          | `GlideClient`, `GlideClusterClient` (Tokio), blocking `sync::Sync*` clients behind the default `sync` feature.                                                                                                                                                                                                        |
| Tests            | 29 integration files (`tests/it_*.rs`) run as standalone/cluster × RESP2/RESP3, plus unit, mock and parity tests. About 17k lines of Rust in `src/`.                                                                                                                                                                  |
| Dependency graph | 247 unique crates with default features. The crates.io `redis` crate is **not** in the graph. AWS SDK (`aws-config`, `aws-sigv4`, …), OpenTelemetry (`opentelemetry-otlp` with tonic and reqwest), `rustls` + `aws-lc-rs`, `lz4` and `zstd` are always compiled, because `glide-core` has no Cargo features for them. |
| Coexistence      | A scratch project depending on both `valkey-glide` (path) and `redis` 1.8.3 builds. Cargo keeps the two `redis`-named libs apart because their package names differ.                                                                                                                                                  |

## Motivation

### Market and ecosystem demand

redis-rs is the default Rust client for the Redis protocol: 136.8M total
downloads, 39.4M in the last 90 days, and 1,661 reverse dependencies on
crates.io. The next alternatives are far smaller (`fred` 12.0M total,
`deadpool-redis` 16.0M, `bb8-redis` 12.4M, both of which wrap redis-rs).
Valkey users on ElastiCache, MemoryDB, Memorystore and self-hosted clusters
reach Valkey from Rust through this stack today.

### Competitive and parity gap

redis-rs follows Redis, and Redis and Valkey are diverging. GLIDE's own
migration guide already lists redis-rs features with no Valkey equivalent:
`IFDEQ`/`IFDNE` digest comparisons, `XADD` deletion policies and idempotent
production, `XREADGROUP ... CLAIM`, and the hotkeys commands. redis-rs 2.0
(now at 2.0.0-rc.4) continues in that direction. Valkey-only behavior does not
have a natural home there.

GLIDE's other bindings (Python, Java, Node, Go) share `glide-core` features
that a Rust user can only get through GLIDE: AZ affinity, IAM auth for
ElastiCache/MemoryDB, PubSub auto-resubscription, cluster scan, multi-slot
command splitting, OpenTelemetry, compression and client-side caching.

### Strategic alignment

- Cross-language consistency: the Rust client exercises the same core as every
  other binding, so core bugs found by Rust users fix every binding.
- Performance: the Rust binding is the only one with no FFI or socket hop, and
  commands are handed to `glide-core` by value (zero extra payload copies).
- Ownership: GLIDE controls its command surface and release cadence instead of
  depending on a Redis-led project's roadmap.

### Direct user signal

- [#6915](https://github.com/valkey-io/valkey-glide/issues/6915) GA epic and
  [#6519](https://github.com/valkey-io/valkey-glide/issues/6519) preview.
- [#2710](https://github.com/valkey-io/valkey-glide/issues/2710): binary size
  matters for Lambda and small containers.
- [#7169](https://github.com/valkey-io/valkey-glide/issues/7169): an untriaged
  external user report against the Rust client.

### Current workarounds

Rust users either stay on redis-rs (accepting Redis-shaped APIs and no GLIDE
features) or adopt the 0.x preview and work around open breaking bugs (see
Appendix B).

### Cost of inaction

GLIDE remains absent from the largest Valkey-adjacent Rust ecosystem, and the
redis-rs fork inside `glide-core` keeps accumulating maintenance cost with no
user-facing return in Rust.

## What "does not need redis-rs" means

The request can mean four different things. This RFC treats them as separate
levels with separate guarantees.

| Level              | Meaning                                                                                                         | Status                                                      | Proposal                                                                    |
| ------------------ | --------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------- | --------------------------------------------------------------------------- |
| **L1: dependency** | The crates.io `redis` crate is not in a GLIDE user's dependency graph.                                          | Already true.                                               | Keep true; enforce in CI.                                                   |
| **L2: API**        | No redis-rs type, trait or error appears in GLIDE's public API, docs or error messages.                         | True for the public API. Not yet guarded.                   | Guarantee for 1.0; enforce in CI.                                           |
| **L3: code**       | No code derived from redis-rs runs inside GLIDE.                                                                | False. `glide-core-engine` is redis-rs 0.25.2 code.         | Staged replacement (see "Engine independence"). Not a blocker for Rust 1.0. |
| **L4: ecosystem**  | Libraries built on redis-rs traits (`deadpool-redis`, `bb8-redis`, session stores, job queues) work with GLIDE. | False by design. GLIDE does not implement `ConnectionLike`. | Not planned. No compat crate; downstream crates add GLIDE backends instead. |

The GA promise is **L1 + L2**: a Rust project can talk to Valkey through GLIDE
with no `redis` crate, no `redis::` types in its code, and migration docs for
every call site it used. **L3** is delivered in stages after that (see
"Engine independence"); its first two stages land before Rust 1.0 because
they are cheap and make the later stages possible.

## Design Considerations

### Compatibility posture: redis-rs-shaped, GLIDE-authoritative

The client mirrors redis-rs's command surface to make migration mechanical.
That choice has costs:

- redis-rs is a moving target. 1.7.0 has typed returns that cannot represent
  their replies (`hmget` → `Vec<String>`, `lmove` → `String`, …); 2.0 fixes
  them with breaking changes ([#7641](https://github.com/valkey-io/valkey-glide/issues/7641)).
- Some redis-rs features target Redis-only commands and will never exist in
  Valkey.

Proposal:

1. Retarget the parity guard to redis-rs **2.0** before GLIDE 1.0, so GLIDE
   1.0 does not freeze 1.7's known-broken return types.
2. **Freeze the parity snapshot at redis-rs 2.0.** After GLIDE 1.0 the
   snapshot is a migration aid, not a contract, and it does not track later
   redis-rs releases. GLIDE semver governs; a redis-rs release never forces a
   GLIDE change. New commands are added on GLIDE's own schedule, following
   Valkey and the other GLIDE bindings.
3. Every deliberate difference stays in `differences.json` with a reason, and
   `migration.md` documents it.
4. redis-rs-shaped types are **GLIDE-owned copies**, never re-exports of fork
   types. This applies to [#7678](https://github.com/valkey-io/valkey-glide/issues/7678)
   (`Route`/`SlotType`): matching redis-rs's shapes and constructors is fine,
   but use Valkey terminology where it differs (`Primary`, not `Master`) and
   keep validation such as a `Slot` newtype that rejects values ≥ 16384.

### The fork's lib name

`glide-core-engine` compiles as a library named `redis`. Users never import it,
but the name shows up in `glide-core` rustdoc, compiler diagnostics that cross
crate boundaries, `cargo tree` reasoning and backtraces, which undercuts the
"no redis-rs" message. Renaming the lib to `glide_core_engine` is an internal
change to `glide-core`, every binding's Rust glue and the fork's own tests.
Proposed for 0.11, together with the cleanup in
[#2710](https://github.com/valkey-io/valkey-glide/issues/2710).

### Optional dependencies

Every user pays for IAM (AWS SDK), OpenTelemetry (tonic + reqwest),
compression and `aws-lc-rs`, even if they use none of them. That conflicts
with Rust norms and with [#2710](https://github.com/valkey-io/valkey-glide/issues/2710).
Gating requires Cargo features in `glide-core` and `glide-telemetry`, which
every binding consumes. The bindings enable all features, so their behavior
does not change; only the Rust crate exposes the choice. See the feature table
in the Specification.

### Runtime

`glide-core` is built on Tokio. Runtime-agnostic support (smol, async-std)
would require abstracting timers, spawning and I/O throughout the core and the
fork. async-std is discontinued and the remaining demand is small, so Tokio
stays the only supported runtime and the docs say so.

### Pooling and dedicated connections

GLIDE multiplexes commands over one connection per node. redis-rs users who
reach for `deadpool-redis` or `bb8-redis` generally do not need a pool with
GLIDE, but blocking commands (`BLPOP`, `XREAD BLOCK`), `WATCH`/`MULTI` and
`MONITOR` need dedicated connections. This is tracked in
[#6917](https://github.com/valkey-io/valkey-glide/issues/6917) and
[#5815](https://github.com/valkey-io/valkey-glide/issues/5815) (RFC: Dedicated
Connection Pooling). The Rust client follows that RFC rather than designing
its own pool.

### Cross-language rollout

This RFC is Rust-first, but four parts touch shared code:

- Cargo features in `glide-core`/`glide-telemetry` (default-on, no behavior
  change for other bindings).
- Renaming the fork's lib (mechanical `redis::` → `glide_core_engine::` in
  every crate that uses it).
- Engine independence: stage 1 moves every binding's imports to the
  `glide_core::protocol` facade; stages 2–4 replace engine internals and run
  every binding's suites on both implementations.
- The lower MSRV applies to `glide-core`, which every binding builds.

Core bugs found through Rust (for example
[#7673](https://github.com/valkey-io/valkey-glide/issues/7673),
[#7683](https://github.com/valkey-io/valkey-glide/issues/7683),
[#7653](https://github.com/valkey-io/valkey-glide/issues/7653)) are fixed in
`glide-core` and benefit all bindings.

### Server compatibility

Unchanged from GLIDE: the Supported Engine Versions table in the root
README. Commands newer than Redis 6.2 document their minimum version
([#7402](https://github.com/valkey-io/valkey-glide/issues/7402)).

### Alternatives considered

| Alternative                                             | Why not                                                                                                                       |
| ------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| Depend on crates.io `redis` and add GLIDE on top        | Fails L1. Inherits Redis-shaped roadmap and the copy cost of `ConnectionLike`.                                                |
| Re-export fork types (`redis::Value`, `RedisError`)     | Fails L2. Ties GLIDE's public API to a 0.25.2 fork that cannot change freely.                                                 |
| Implement redis-rs's `ConnectionLike` in the main crate | Fails L1 (needs the `redis` crate) and forces a payload copy per command (see `DESIGN.md`).                                   |
| GLIDE-native API with no redis-rs resemblance           | Highest migration cost for the largest pool of potential users. Mirroring is cheap because the table and guard already exist. |
| Replace the fork before Rust 1.0                        | Very large, affects every binding, and Rust users cannot observe it once L2 holds. Staged after 1.0 instead (stages 2–4).     |
| Rewrite the engine in one step                          | One very large change across every binding with no way to compare against the old engine. Staged replacement instead.         |
| Switch the engine to upstream redis-rs 2.x or `fred`    | Fails L1 and gives up GLIDE's engine changes (147 commits since vendoring). Neither has GLIDE's cluster and AZ logic.         |

## Specification

### Scope

**In scope:**

- L1 and L2 guarantees, enforced in CI.
- Public API freeze for 1.0: naming ([#7702](https://github.com/valkey-io/valkey-glide/issues/7702)),
  option-type layout ([#7385](https://github.com/valkey-io/valkey-glide/issues/7385)),
  routing types ([#7678](https://github.com/valkey-io/valkey-glide/issues/7678)),
  public `Cmd` constructors and `Pipeline::add_command` ([#7292](https://github.com/valkey-io/valkey-glide/issues/7292)).
- redis-rs 2.0 parity target ([#7641](https://github.com/valkey-io/valkey-glide/issues/7641)),
  missing commands ([#7060](https://github.com/valkey-io/valkey-glide/issues/7060))
  and options ([#7082](https://github.com/valkey-io/valkey-glide/issues/7082)).
- First-party typed APIs for every user-facing Valkey 9.2 command and for
  valkey-search 1.3, valkey-json 1.1 and valkey-bloom 1.0, so no documented
  command needs `custom_command` (gaps in
  [command-gap-analysis.md](command-gap-analysis.md)).
- GLIDE feature parity with other bindings listed in #6915: client-side
  caching (#6918), circuit breaker (#6916), custom credentials provider
  (#6921), client info tag (#6920), custom logger (#6922), compression
  (#6878), OpenTelemetry command spans (#7699).
- Cargo feature gating for heavy optional dependencies.
- Rename the fork's lib to `glide_core_engine`.
- Engine independence (L3), stages 0–1 before Rust 1.0, stages 2–4 after.
- MSRV lowered to 1.88 for `glide-core` and `valkey-glide`, with a written policy.
- A public test-support story to replace `redis-test`.
- Cargo workspace for published crates ([#7261](https://github.com/valkey-io/valkey-glide/issues/7261)).

**Out of scope:**

- Making Rust 1.0 wait for the full engine replacement.
- Implementing redis-rs traits in `valkey-glide` (L4).
- Sentinel and non-Tokio runtimes.
- A Rust-specific connection pool (follows #5815).

### Public API invariants (1.0)

1. No path in the public API names a type from `glide-core-engine`,
   `glide-core` internals, or the crates.io `redis` crate. Third-party types
   allowed in the public API are limited to `bytes::Bytes`, `num_bigint`
   (#7664), `std` types, and optional-feature types (`rust_decimal`,
   `bigdecimal`, #7663).
2. Every public enum and every configuration or reply struct that can grow is
   `#[non_exhaustive]` (already the practice: 18 occurrences, including
   `GlideError`, `ReadFrom`, `Route`, `GlideClientConfiguration`).
3. Errors: `GlideError` implements `std::error::Error + Send + Sync + 'static`,
   is matchable by variant, and exposes server error codes without string
   parsing (depends on [#7442](https://github.com/valkey-io/valkey-glide/issues/7442)).
   `Display` output never contains `redis` crate names.
4. Clients are `Clone + Send + Sync`, cheap to clone, and take `&self`.
5. Commands return `ValkeyResult<T>`; no panics on server or network input.
   Panics are allowed only for documented programmer errors.
6. One public path per type ([#7385](https://github.com/valkey-io/valkey-glide/issues/7385)).
7. Builder methods use `with_*` ([#7702](https://github.com/valkey-io/valkey-glide/issues/7702)),
   following the Rust API Guidelines (C-BUILDER).
8. Behavioral bugs that need a breaking fix land before 1.0 (Appendix B), so
   1.0 does not ship known-wrong semantics.

### Cargo features

| Feature                      | Default | Pulls in                                          | Notes                                                                                                                                                   |
| ---------------------------- | ------- | ------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `sync`                       | on      | nothing extra                                     | Existing.                                                                                                                                               |
| `tls`                        | on      | `rustls`, `tokio-rustls`                          | TLS connections.                                                                                                                                        |
| `tls-aws-lc`                 | on      | `aws-lc-rs`                                       | rustls provider. Needs a C toolchain and CMake on some targets.                                                                                         |
| `tls-ring`                   | off     | `ring`                                            | Alternative provider for targets where `aws-lc-sys` is hard to build. Mutually exclusive with `tls-aws-lc` at runtime; pick one provider at build time. |
| `iam`                        | on      | `aws-config`, `aws-sigv4`, `aws-credential-types` | `ServerCredentials::iam`.                                                                                                                               |
| `telemetry`                  | on      | `glide-telemetry`, `opentelemetry*`               | `glide::telemetry`. Consider splitting exporters (`telemetry-grpc`, `telemetry-http`).                                                                  |
| `compression`                | on      | `lz4`, `zstd`                                     | #6878.                                                                                                                                                  |
| `num-bigint`                 | on      | `num-bigint`                                      | Matches redis-rs's default (#7664).                                                                                                                     |
| `rust_decimal`, `bigdecimal` | off     | the respective crate                              | Matches redis-rs (#7663).                                                                                                                               |
| `test-util`                  | off     | nothing extra                                     | Public mock/test helpers.                                                                                                                               |

All features marked "on" stay default-on at 1.0, so a default build matches
the other GLIDE bindings and existing users see no change. Users who want a
smaller build opt out with `default-features = false` and re-enable what they
need. Features are additive; no feature removes API. CI builds and
lints `--no-default-features`, each feature alone, and `--all-features`
(today CI never builds `--no-default-features`, per `rust/AGENTS.md`).

### Blocking (`sync`) layer

- Keep the process-wide multi-thread runtime.
- Calling a blocking method from inside a Tokio runtime makes Tokio panic
  ("Cannot start a runtime from within a runtime"). Detect this with
  `tokio::runtime::Handle::try_current()` and return
  `GlideError::Configuration` with a message pointing to the async client,
  instead of panicking.
- Document that blocking clients must not be shared with async code.

### Client lifecycle: `close()`

Today neither Rust client has a close method. A client shuts down when its
last clone is dropped: `glide-core`'s background tasks hold `Weak` references,
so they stop once the last `Arc` goes away. That works, but has gaps:

- Clients are cheap clones. Dropping one handle closes nothing, and no single
  owner can shut the client down for every clone.
- Rust has no async `Drop`, so a caller cannot await a graceful shutdown
  (in-flight requests, unsubscribes, connection close) before the process
  exits, a Lambda freezes, or a `#[tokio::test]` runtime is torn down.
- `GlideError::Closing` exists but no code path returns it.
- Python, Java, Node and Go all expose `close()`.

Add an explicit close, and keep `Drop` as the fallback:

```rust,ignore
impl GlideClient {
    /// Closes the client for every clone. Idempotent.
    pub async fn close(&self);
    /// Whether `close` has been called on any clone.
    pub fn is_closed(&self) -> bool;
}
// Same on GlideClusterClient; SyncGlideClient::close(&self) blocks.
```

Semantics:

1. `close` takes `&self`, not `self`. Consuming `self` would only drop one
   clone and would not close the others.
2. It sets a closed flag shared by all clones. Requests in flight and any
   request after it return `GlideError::Closing`, matching the other bindings'
   `ClosingError`.
3. It stops background tasks (reconnect, periodic topology checks, pub/sub
   resubscription) and closes connections, then returns. A second call returns
   immediately.
4. Closing is final; there is no reopen. Create a new client instead.
5. It does not shut down global OpenTelemetry exporters, which are
   process-wide and may be shared with other clients.
6. Dropping every clone without calling `close` still releases everything, as
   today.

This needs a close entry point in `glide-core` (a closed flag in
`ClientShared` checked on dispatch, and replacing the inner client wrapper so
connections drop). The FFI bindings could use the same entry point. Adding
`close` is additive and can land in a 0.x minor.

### Engine independence (L3)

Goal: retire `glide-core-engine` so no redis-rs-derived code runs in GLIDE.
The work is staged so each stage ships on its own, can be rolled back, and
never blocks a binding release.

**What is being replaced.** The fork is about 42k lines of Rust in `src/` plus
about 23.6k lines of tests, nearly the size of `glide-core` itself (about 43k).
It has diverged a lot from upstream: 147 commits since it was vendored in
October 2024, and about 5.3k lines in files GLIDE added after that (the
client-side cache, pipeline routing, retry strategies, buffer pool, pub/sub
synchronizer). Licenses: BSD-3-Clause for the crate and MIT for
`cluster_async/` (Atsushi Koge, Markus Westerlind).

| Area                         | Main files (lines)                                                                                                                                                                                                    | Used by                                                   |
| ---------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------- |
| Protocol and values          | `types.rs` (2,560), `parser.rs` (1,239), `cmd.rs` (1,359), `pipeline.rs` (476)                                                                                                                                        | Every crate: `Value`, `Cmd`, `RedisError`, `PushInfo`     |
| Connections                  | `aio/multiplexed_connection.rs` (3,354), `connection.rs` (2,160), `aio/connection.rs` (1,075), `push_manager.rs`, `tls.rs`                                                                                            | `glide-core`                                              |
| Cluster                      | `cluster_async/mod.rs` (7,346), `cluster_routing.rs` (2,541), `cluster_slotmap.rs` (1,941), `connections_container.rs` (1,946), `pipeline_routing.rs` (1,444), `cluster_topology.rs` (1,248), `cluster_scan.rs` (926) | `glide-core`; routing types also in Java, FFI, Node, Rust |
| GLIDE-authored, already ours | `cache/` (3,267), `retry_strategies.rs`, `buf_pool.rs`, `pubsub_synchronizer.rs`                                                                                                                                      | `glide-core`, Python                                      |
| No consumer found            | `sentinel.rs` (777), `aio/connection_manager.rs` (314), `cluster_pipeline.rs` (152), sync `Connection`/cluster paths (to confirm)                                                                                     | none                                                      |

Today five crates outside `glide-core` depend on the fork directly (`ffi`,
`java`, `node/rust-client`, `python/glide-async`, `rust`). They mostly use
`Value`, `Cmd`, `Pipeline`, `RedisError`/`ErrorKind`, `PushInfo`/`PushKind`
and routing types.

**Stages.**

| Stage                  | Work                                                                                                                                                                                                                                                                                                                      | Exit criteria                                                                                                                                         | When            |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- | --------------- |
| 0. Trim and rename     | Delete code with no consumer (sentinel, connection manager, sync pipeline paths, unused `commands` traits), the leftover `dummy-for-ort` workspace, and unused Cargo features ([#2710](https://github.com/valkey-io/valkey-glide/issues/2710)). Rename the lib from `redis` to `glide_core_engine`.                       | All binding CI green. Fork line count and release binary size recorded before and after.                                                              | Rust 0.11       |
| 1. Seal the boundary   | `glide-core` exposes a GLIDE-owned facade (`glide_core::protocol`: `Value`, `Cmd`, `Pipeline`, `PushInfo`, `PushKind`, `ServerError`, routing types). Bindings import only from it, and `glide-core-engine` is removed from every binding's `Cargo.toml`. CI fails if any crate other than `glide-core` names the engine. | Only `glide-core` depends on the engine. The engine can change without touching bindings.                                                             | Before Rust 1.0 |
| 2. Protocol and values | GLIDE-owned RESP2/RESP3 parser and encoder, `Value`, `Cmd` and errors in `glide-core`. Zero-copy and buffer-pool behavior kept.                                                                                                                                                                                           | Differential and fuzz tests pass (below). No regression in `benchmarks/`.                                                                             | After 1.0       |
| 3. Connections         | Multiplexed connection, handshake (`HELLO`, `AUTH`, `SELECT`, `CLIENT SETINFO`), TLS, push handling. Move the GLIDE-authored modules (cache, retries, buffer pool, pub/sub synchronizer) into `glide-core` unchanged.                                                                                                     | Standalone suites of every binding pass on both engines.                                                                                              | After 1.0       |
| 4. Cluster             | 4a: move routing, slot map, topology, `MOVED`/`ASK` handling, connection container, pipeline routing and cluster scan into `glide-core` unchanged, then delete the engine crate. 4b: refactor sub-parts in place (see "Stage 4 options and wrapper impact").                                                              | 4a: every wrapper's suite passes with no glue changes. 4b: cluster, failover and slot-migration suites pass on both implementations, including PHP's. | After 1.0       |

**How each replacement ships.**

- A compile-time Cargo feature in `glide-core` (`engine-next`) selects the new
  implementation for that stage's modules. No runtime switch, so there is no
  per-command cost.
- CI runs every binding's suite on both implementations while a stage is in
  progress. The default flips in one release, the old code is deleted in the
  next.
- The fork's tests (about 23.6k lines) are ported first and act as the
  specification for each stage.
- Parser and encoder: `cargo fuzz` targets, `proptest` round trips
  (encode → parse → equal), and a differential test feeding the same byte
  streams to both parsers, including partial frames.
- Connections and cluster: the existing failover, slot-migration and
  reconnection tests, plus a differential harness that sends the same command
  sequence through both engines against one server and compares replies.
- Benchmarks (`benchmarks/`, `glide-core/benches`) must show no regression in
  p50/p99 latency, throughput or memory before a default flips.
- `unsafe` follows the existing core rules; new `unsafe` needs a `// SAFETY:`
  comment and Miri coverage where `ffi/miri-tests` can reach it.

**Ownership.** `glide-core` owns all engine stages: design, reviews, CI on
both implementations, and the decision to flip each default. Binding owners
review stage 1 (their imports move to the facade) and sign off before each
default flip, because their suites are the acceptance tests.

#### Stage 4 options and wrapper impact

Every wrapper reaches the cluster logic through `glide-core`, but the
contract is wider than the `Client` API. Four things cross into the wrappers:

1. **Routing types.** Route conversion (`RoutingInfo`,
   `SingleNodeRoutingInfo`, `MultipleNodeRoutingInfo`, `SlotAddr`,
   `ResponsePolicy`, `get_slot`) is written four times: `ffi/src` (19
   `RoutingInfo` uses), `java/src` (13), `node/rust-client/src` (13) and
   `glide-core/src/socket_listener.rs` (11).
2. **Cluster scan cursors.** `ScanStateRC` handles live in
   `glide-core/src/cluster_scan_container.rs` and are released differently
   per wrapper (finalizers in Java and Node, explicit free through FFI in Go
   and PHP, object teardown in Python).
3. **Error classification.** `glide_core::errors::error_type` maps the fork's
   `ErrorKind`, `is_timeout()` and `is_unrecoverable_error()` to
   `RequestErrorType` (`Unspecified`, `ExecAbort`, `Timeout`, `Disconnect`,
   `CircuitBreakerOpen`). Every wrapper maps that to its own exceptions, and
   Java's glue also builds `ErrorKind::ClientError` errors directly (29 uses).
4. **Behavior.** `MOVED`/`ASK` handling, topology refresh, AZ affinity,
   `ReadFrom`, multi-slot splitting and response aggregation are visible in
   every wrapper's results, timings and error text.

| Wrapper      | Path to `glide-core`                                                                        | Uses fork cluster types in its own glue | Cluster test files / error-text assertions | Move-then-refactor               | Rewrite                                                                                        |
| ------------ | ------------------------------------------------------------------------------------------- | --------------------------------------- | ------------------------------------------ | -------------------------------- | ---------------------------------------------------------------------------------------------- |
| Python async | PyO3 `glide-async` + `glide-core` socket layer (protobuf)                                   | No (conversion is in `socket_listener`) | 69 / 33 (shared Python tests)              | Rebuild only                     | `socket_listener` conversions; error mapping must match                                        |
| Python sync  | CFFI into `ffi`                                                                             | No (through `ffi`)                      | same suite                                 | Rebuild only                     | `ffi` conversions; cursor free path                                                            |
| Node         | NAPI `node/rust-client` (protobuf)                                                          | Yes                                     | 23 / 12                                    | Rebuild only                     | Rewrite route and cursor glue; finalizer semantics must hold                                   |
| Java         | JNI `java/src` (protobuf)                                                                   | Yes, plus direct `ErrorKind` use        | 27 / 10                                    | Rebuild only                     | Rewrite route glue and error construction; exception mapping must match                        |
| Go           | CGO into `ffi` (`libglide_ffi`)                                                             | No (through `ffi`)                      | 39 / 2                                     | Rebuild only                     | `ffi` conversions; explicit cursor free                                                        |
| PHP          | C extension, static `libglide_ffi.a` from a pinned `valkey-glide` submodule (separate repo) | No (through `ffi`)                      | separate repo                              | Picked up on next submodule bump | Same as Go, but our CI does not run PHP's suite; needs a cross-repo job before a default flips |

Stage 1's `glide_core::protocol` facade takes care of item 1 for both options,
as long as the facade types stay stable. Items 2–4 are what a rewrite puts at
risk.

**Recommendation: move, then refactor.**

- **4a. Move.** Move the cluster modules from the engine crate into
  `glide-core` (for example `glide-core/src/cluster/`) with no behavior
  change, then delete `glide-core-engine`. Wrappers rebuild with no glue
  changes, and PHP gets it on its next submodule bump. This ends the separate
  crate quickly.
- **4b. Refactor in place.** Replace sub-parts only when there is a reason
  (a bug class, performance, or a maintenance cost), each behind
  `engine-next` with a differential test against the moved code: slot map
  and topology refresh first, then the connection container, with
  `cluster_async/mod.rs` (7,346 lines) last.
- The cost of this order is that adapted code and its MIT/BSD notices remain
  longer. A full rewrite would remove more of it, but changes items 2–4 for
  six wrappers at once and needs PHP's suite running in our CI first.

**After stage 4.** `glide-core-engine` stops receiving releases and its
crates.io page points to `glide-core`. Published versions are not yanked.
Attribution: the BSD-3-Clause and MIT notices stay in `THIRD-PARTY-NOTICES`
for as long as any code adapted from the fork remains, which in practice
means indefinitely; a rewrite done while reading the old code is still
adapted code.

### MSRV policy

The current 1.94.1 comes only from the newest AWS SDK crates. The highest
`rust-version` among the other normal dependencies is 1.88 (`tonic`), and
edition 2024 needs 1.85.

Cargo's [MSRV guidance](https://doc.rust-lang.org/cargo/reference/rust-version.html)
expects every feature to work on the declared version, but only requires each
dependency's version range to include _at least one_ compatible release.
Edition-2024 projects resolve with `incompatible-rust-versions = "fallback"` by
default, so on an older toolchain Cargo picks older releases that still fit.
`aws-config = "1"` includes `aws-config` 1.8.13 (February 2026, MSRV 1.88).

Verified on a scratch copy with Rust 1.88.0 and a fallback-resolved lockfile
(`aws-config` 1.8.13, `aws-sigv4` 1.3.8, `tonic` 0.14.9):

- `glide-core`: `cargo check --lib` with default features passes.
- `valkey-glide`: `cargo check --all-features --all-targets` passes with
  `rust-version = "1.88"`, and all 320 unit tests (`cargo test --lib
--all-features`) pass.

- `valkey-glide` integration tests (`cargo integration-tests --all-features`)
  against Valkey 9.2.0: 1,221 passed, 0 failed, 0 ignored across all 29
  `tests/it_*.rs` files, covering standalone and cluster, RESP2 and RESP3,
  TLS, auth, compression, pub/sub and telemetry. Runtime about 5
  minutes.

Not yet verified on 1.88: the `glide-core`, `glide-telemetry` and
`glide-logger` test suites, and the binding crates (which are unpublished and
keep their own toolchains).
Users on 1.88 who enable `iam` get `aws-config` 1.8.x rather than the newest
SDK. That is their trade-off to make, and `cargo update` brings the newest SDK
back on a newer toolchain.

Proposal:

1. Set `rust-version = "1.88"` on `valkey-glide`, `glide-core`,
   `glide-telemetry` and `glide-logger`. `glide-core-engine` keeps 1.71 until
   stage 0 aligns it.
2. Policy: support the Rust release from about 12 months earlier or newer.
   Raising `rust-version` is a minor version bump with a CHANGELOG entry,
   never a patch.
3. Version ranges stay broad (`aws-config = "1"`) so fallback resolution keeps
   working. Raise a lower bound only when GLIDE needs an API that older
   releases lack, and only if a release meeting the MSRV still satisfies it.
4. CI adds an MSRV job: install the declared toolchain, generate a fresh
   lockfile with `CARGO_RESOLVER_INCOMPATIBLE_RUST_VERSIONS=fallback`, then
   build, lint and run unit tests with `--all-features`. The normal jobs keep
   using the committed lockfile on the pinned newer toolchain.
5. `clippy::incompatible_msrv` stays on so new code cannot use std APIs newer
   than the MSRV.

### Connection URLs

`from_url` accepts `redis://`, `rediss://`, `valkey://`, `valkeys://` and
`?protocol=resp2|resp3` ([#7031](https://github.com/valkey-io/valkey-glide/issues/7031)),
plus `unix://` for standalone ([#7645](https://github.com/valkey-io/valkey-glide/issues/7645)).
GLIDE owns the parser. It must not rely on the fork's `IntoConnectionInfo`
so that URL behavior can change without touching the fork.

### Testing support (replacing `redis-test`)

redis-rs users rely on `redis-test` (`MockRedisConnection`, 2.6M downloads).
Under `test-util`, expose:

- A mock executor that returns scripted `ValkeyValue` replies for given `Cmd`s,
  usable anywhere a client is accepted generically (`C: AsyncCommands`).
- Assertion helpers for encoded arguments (`assert_args`, already internal in
  `src/test_utils.rs`).

This requires the sealed dispatch traits (`CommandDispatch`) to accept a
GLIDE-provided mock implementation without unsealing them for users.

### CI gates

| Gate                                                                                                                                                | Purpose                                 |
| --------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------- |
| `cargo tree -e normal -i redis` must fail (no such package) for default, none and all features                                                      | L1                                      |
| Public API snapshot (`cargo public-api`) checked in; CI fails on any path containing `redis` or `glide_core_engine`, and on unreviewed diffs        | L2 and API review                       |
| `cargo semver-checks` against the last release (from 1.0)                                                                                           | Semver discipline                       |
| Feature matrix build and clippy (`--no-default-features`, each feature, `--all-features`)                                                           | Additive features                       |
| MSRV build with `rust-version` toolchain ([#7175](https://github.com/valkey-io/valkey-glide/issues/7175))                                           | MSRV is real                            |
| docs.rs-equivalent build with `--cfg docsrs` and `#[doc(cfg(...))]` on gated items ([#7297](https://github.com/valkey-io/valkey-glide/issues/7297)) | Docs per feature                        |
| `cargo deny check` (existing)                                                                                                                       | Licenses and advisories                 |
| `cargo +nightly update -Z minimal-versions && cargo check`                                                                                          | Lower bounds in `Cargo.toml` are honest |
| Existing parity guard, retargeted to 2.0 and extended ([#7288](https://github.com/valkey-io/valkey-glide/issues/7288))                              | Migration accuracy                      |

### Success Criteria

- L1 and L2 enforced by CI on every PR.
- All issues labeled `breaking` + `Rust` (Appendix B) closed before 1.0.
- 100% of redis-rs 2.0 command-table methods either implemented or listed in
  `differences.json` with a reason.
- A no-default-features build has at least 40% fewer crates than today's 247
  (target to confirm by measurement in phase 2).
- Latency and throughput at least equal to redis-rs multiplexed connections on
  the existing benchmark harness (`benchmarks/rust`) for GET/SET, MGET across
  slots and pipelines.
- External signal: crates.io downloads trend, first downstream crates
  (session stores, rate limiters) with a GLIDE backend, issue reports from
  users outside the maintainer group.

### Benchmarking

Add the Rust client to `benchmarks/` next to the other bindings and compare
against redis-rs 1.x/2.x `MultiplexedConnection` and `cluster_async` with the
same workload: GET/SET 100 B and 16 KB, 90/10 read/write, pipelines of 100,
cluster MGET over 3 and 9 shards. Track p50/p99 latency, throughput, RSS, and
release binary size with default and no default features.

### Rollout Plan

| Phase             | Release        | Work                                                                                                                                                                                                                                                                                           |
| ----------------- | -------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1. Guardrails     | 0.10.x         | CI gates for L1/L2, feature-matrix build, MSRV job on 1.88 with fallback resolution, docs.rs build. Lower `rust-version` to 1.88. No API change.                                                                                                                                               |
| 2. Breaking batch | 0.11           | Fix all breaking bugs (Appendix B), API naming and layout (#7702, #7385, #7678, #7691), public `Cmd` constructors (#7292), redis-rs 2.0 parity (#7641), engine stage 0 (trim and lib rename), Cargo features in `glide-core`. One CHANGELOG section listing every break with a migration line. |
| 3. Parity         | 0.12–0.x       | Missing commands and options (#7060, #7082, #7286), GLIDE features (#6916–#6922, #6878, #7699), `test-util`, Unix sockets, explicit `close()`, engine stage 1 (`glide_core::protocol` facade), workspace (#7261).                                                                              |
| 4. RC and GA      | 1.0.0-rc → 1.0 | Bug bash, benchmark report, refreshed docs (#7205, #7295), migration guide for redis-rs 1.x and 2.x, published examples under `examples/rust`.                                                                                                                                                 |
| 5. Engine         | 1.x minors     | Engine stages 2–4 behind `engine-next`, one stage at a time, default flipped per stage after both implementations pass every binding's suites. No public API change.                                                                                                                           |

Documentation at GA: README quick starts, `migration.md` (redis-rs 1.x and
2.x), a "Valkey-only features" page, feature-flag table, and docs at
glide.valkey.io with Rust as a selectable language.

### Open Questions

Resolved:

- `iam`, `telemetry` and `compression` stay default-on at 1.0.
- The parity snapshot freezes at redis-rs 2.0.
- No L4 compat crate.
- Explicit `close()`: proposed above in "Client lifecycle"; pending review.
- MSRV: lowered to 1.88 for the published crates (see "MSRV policy").
- Engine replacement is part of this RFC (see "Engine independence").

- Ownership: the engine work belongs to `glide-core`, not the Rust client.

Open:

1. Stage 4 (cluster): move-then-refactor (recommended) or rewrite? See
   "Stage 4 options and wrapper impact".

## Caveats

These are the points most likely to surprise a team adopting this plan.

1. **"No redis-rs" is true at the API, not inside.** The engine is a fork of
   redis-rs 0.25.2 (BSD-3-Clause). Security audits, SBOMs and license scans
   will find redis-rs-derived code under the name `glide-core-engine`. The
   attribution and BSD license must stay with it.
2. **Ecosystem crates will not work.** `deadpool-redis`, `bb8-redis`,
   `tower-sessions`/`actix-session` Redis stores, rate limiters and job
   queues that take a redis-rs connection need a GLIDE backend or a port.
   This is the main migration cost for real applications, more than call-site
   renames.
3. **Migration is mostly, not fully, mechanical.** Type renames
   (`RedisResult` → `ValkeyResult`, `FromRedisValue` → `FromValkeyValue`, …),
   method renames in custom trait impls, `&mut conn` → `&client`, no
   `IntoConnectionInfo`, no `ErrorKind`, several typed returns that differ,
   and scan iterators that yield `Result` per item. Generic code bounded on
   `ConnectionLike` must be re-bounded.
4. **Chasing redis-rs has a ceiling.** Some redis-rs APIs target Redis-only
   commands. The more Valkey and Redis diverge, the more `differences.json`
   grows and the less "source-compatible" means.
5. **Build cost.** First builds compile ~247 crates including `aws-lc-sys`
   (C, needs CMake/clang on some platforms) until the feature work lands.
   Cross-compilation (musl, Windows, macOS) is untested; CI covers Linux
   x86_64 and aarch64 only.
6. **MSRV relies on fallback resolution.** 1.88 matches redis-rs only because
   Cargo can pick older AWS SDK releases. Users on 1.88 get an older SDK, a
   future `aws-config` 1.x could need an API GLIDE starts using, and a project
   with an old committed lockfile or resolver `"2"` must resolve on its own.
   The `valkey-glide` unit and integration suites pass on 1.88; the other crates' suites are not yet verified.
7. **Tokio only.** The blocking layer runs a hidden global Tokio runtime; it
   panics today if called from inside an async context.
8. **0.x churn.** The 13 breaking issues in Appendix B are still open. Early adopters
   should pin `=0.10.x` and expect a migration step at 0.11.
9. **Multiplexing semantics.** One connection per node means `WATCH`,
   blocking commands, `MONITOR` and per-connection state (`SELECT`,
   `CLIENT SETNAME`) behave differently from a redis-rs pooled connection
   until dedicated connections (#5815, #6917) land.
10. **Shared-core changes ripple.** Feature gating and the lib rename touch
    `glide-core`, which Java, Python, Node and Go also build. Those changes
    need those bindings' CI and reviewers.
11. **Monorepo coupling.** The published crates resolve by version from
    crates.io, but contributors need the whole monorepo, and five crates must
    be published in order with matched versions until #7261 lands.
12. **Engine replacement is large and slow.** About 42k lines of engine code and 23.6k lines of tests, used by every binding. It spans several releases after Rust 1.0, and while a stage is in progress CI runs every suite twice. Notices (BSD-3-Clause and MIT for `cluster_async/`) stay even after the rewrite.

## Appendix

### A. Rust best practices applied

- Rust API Guidelines: `C-BUILDER` (`with_*`), `C-COMMON-TRAITS`, `C-GOOD-ERR`
  (`Error + Send + Sync`), `C-STRUCT-PRIVATE`, `C-SEALED` (dispatch traits),
  `C-FEATURE` (additive features), `C-STABLE` (no unstable public deps).
- `#[non_exhaustive]` for growable enums/structs; sealed traits for
  extension points GLIDE must control.
- `#![forbid(unsafe_code)]` in the user-facing crate; unsafe stays in core/FFI.
- Clippy `await_holding_lock = deny`; extend with `clippy::pedantic` review
  for public items before 1.0.
- `cargo-semver-checks`, `cargo public-api`, `cargo deny`, minimal-versions,
  MSRV CI, docs.rs `doc(cfg)`.
- Committed `Cargo.lock` for reproducible CI; lockfile is ignored by
  downstream users as usual for libraries.

### B. Open breaking issues to land before 1.0

| Issue                                                          | Summary                                                                                                            |
| -------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| [#7665](https://github.com/valkey-io/valkey-glide/issues/7665) | Inconsistent error types for invalid config (PR [#7715](https://github.com/valkey-io/valkey-glide/pull/7715))      |
| [#7672](https://github.com/valkey-io/valkey-glide/issues/7672) | `WATCH`-aborted `exec` looks like an empty batch (PR [#7714](https://github.com/valkey-io/valkey-glide/pull/7714)) |
| [#7673](https://github.com/valkey-io/valkey-glide/issues/7673) | Cluster batches ignore multi-node routes (PR [#7677](https://github.com/valkey-io/valkey-glide/pull/7677))         |
| [#7686](https://github.com/valkey-io/valkey-glide/issues/7686) | Subscribe returns before server confirmation                                                                       |
| [#7688](https://github.com/valkey-io/valkey-glide/issues/7688) | `pubsub_numsub` pairs in random order                                                                              |
| [#7691](https://github.com/valkey-io/valkey-glide/issues/7691) | Sharded pub/sub accepted on standalone                                                                             |
| [#7701](https://github.com/valkey-io/valkey-glide/issues/7701) | OTel exporters configured by endpoint strings                                                                      |
| [#7702](https://github.com/valkey-io/valkey-glide/issues/7702) | `with_*` builder naming (PR [#7723](https://github.com/valkey-io/valkey-glide/pull/7723))                          |
| [#7362](https://github.com/valkey-io/valkey-glide/issues/7362) | Typed multi-node server commands fail on cluster                                                                   |
| [#7385](https://github.com/valkey-io/valkey-glide/issues/7385) | Option type location and single export path                                                                        |
| [#7641](https://github.com/valkey-io/valkey-glide/issues/7641) | redis-rs 2.0 parity target                                                                                         |
| [#7678](https://github.com/valkey-io/valkey-glide/issues/7678) | Routing types shaped like redis-rs                                                                                 |
| [#6872](https://github.com/valkey-io/valkey-glide/issues/6872) | Restrict `SCAN` on cluster clients                                                                                 |

Related non-breaking bugs: #7653 (lazy connect error, PR #7685), #7683 (pub/sub
with RESP2), #7633 (deprecated ZRANGE variants, PR #7647), #7815 (empty AZ
accepted), #7169 ("Connection in recovery" not retryable).

### C. Other relevant open PRs

#7843 (`DELEX`), #7874 (`HSETEX` NX/XX), #7679 (RESP3 normalization docs),
#7451 (Unix sockets, draft), #7752 and #7597 (per-node `INFO`, draft),
#7181 (bump `num-bigint` to 0.5, relates to #7664).

### D. References

- [implementation-plan.md](implementation-plan.md): ordered work items,
  dependencies and estimates for this RFC
- [command-gap-analysis.md](command-gap-analysis.md): typed-command gaps
  against Valkey 9.2, valkey-search 1.3, valkey-json 1.1, valkey-bloom 1.0
  and redis-rs 1.7
- `rust/README.md`, `rust/DESIGN.md`, `rust/migration.md`,
  `rust/docs/redis-rs-compatibility.md`, `rust/docs/dev/options.md`
- redis-rs 2.0 migration guide: https://github.com/redis-rs/redis-rs/blob/main/version2.md
- Rust API Guidelines: https://rust-lang.github.io/api-guidelines/
- crates.io figures retrieved from the crates.io API at the time of writing.
