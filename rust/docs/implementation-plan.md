# Implementation plan: Valkey GLIDE for Rust

Agent hand-off version with TDD briefs: [agent-tasks.md](agent-tasks.md).

Turns [rfc-redis-rs-free-rust-client.md](rfc-redis-rs-free-rust-client.md)
(the "what" and "why") and [command-gap-analysis.md](command-gap-analysis.md)
(the command gaps) into ordered, reviewable work items.

**Verified against:** upstream `main` at `a19098e04`, Valkey 9.2.0,
`valkey/valkey-bundle:9.2.0` (valkey-json 1.1.0, valkey-bloom 1.0.1,
valkey-search 1.3.0), and the state of every issue and PR listed below at
the time of writing. Re-check issue and PR state before starting each work
item; anything listed as open may have merged.

## Ground rules

These apply to every work item.

1. **One work item, one PR**, unless the item says otherwise. Branch from
   current `upstream/main`.
2. **Commits:** Conventional Commits with the `rust` scope (`core` for
   `glide-core`), DCO sign-off (`git commit -s`). Breaking changes use `!`
   (for example `fix(rust)!: …`).
3. **CHANGELOG:** add a line under `## Pending …` in the root `CHANGELOG.md`,
   in the existing style (`* Rust: … ([#NNNN](…))`), under Features, Fixes
   or Breaking Changes.
4. **Quality gate** (from `rust/AGENTS.md`), run in `rust/`:

    ```bash
    cargo fmt --all -- --check
    cargo clippy --all-features --all-targets -- -D warnings
    cargo clippy --all-targets -- -D warnings
    cargo deny --config ../deny.toml check
    RUSTDOCFLAGS="-Dwarnings" cargo doc --no-deps --document-private-items
    cargo unit-tests && cargo doc-tests && cargo integration-tests
    ```

    Integration tests need `valkey-server` on `PATH` (or
    `VALKEY_SERVER_PATH`), a `valkey-cli`/`redis-cli` on `PATH`, and
    `python3`. A dependency change must ship an
    updated `rust/Cargo.lock`, since CI runs with `--locked`.

5. **`glide-core` changes** also run that crate's lint and test gate (see
   `glide-core/AGENTS.md`) and trigger every binding's CI (`glide-core.yml`,
   `ffi.yml`, `go.yml`, `java.yml`, `node.yml`, `python.yml`,
   `python-windows.yml`, `rust.yml`). A
   `glide-core` reviewer is required.
6. **New commands and options** follow `rust/docs/dev/options.md`: implement
   `ToValkeyArgs` (and `ToSingleValkeyArg` only when every value writes
   exactly one argument), add an encoding test covering every variant,
   doc-comment every type and variant, and state the minimum Valkey version
   for anything newer than Redis 6.2 (#7402).
7. **Tests** for new commands use `matrix_test!` (standalone and cluster,
   RESP2 and RESP3) and skip on servers older than the command.
8. **Parity guard:** any change to the command table in
   `src/commands/core.rs` updates `src/parity_tests/differences.json`.

## Team and ownership

| Role                                 | Owns                                                                  |
| ------------------------------------ | --------------------------------------------------------------------- |
| Rust client (2 engineers)            | `rust/`, `rust.yml`, the new `rust-modules.yml`, docs and examples    |
| `glide-core` (1 engineer, part time) | `glide-core/`, `glide-core/redis-rs/`, engine independence, shared CI |
| Binding owners                       | Review engine stage 1 and sign off before each engine default flips   |

Estimates are working days for one engineer who knows the crate. Calendar
time assumes the staffing above, with M3 and M4 run in parallel.

## Milestones

| Milestone | Release | Goal                                            | Estimate      |
| --------- | ------- | ----------------------------------------------- | ------------- |
| M0        | —       | Environment and branch hygiene                  | 0.5 day       |
| M1        | 0.10.x  | Guardrails in CI; MSRV 1.88; no API change      | 1–1.5 weeks   |
| M2        | 0.11    | All breaking changes in one release             | 4–5 weeks     |
| M3        | 0.12+   | Typed commands: Valkey 9.2, modules, management | 7–9 weeks     |
| M4        | 0.12+   | GLIDE feature parity, engine stage 1            | 5–6 weeks     |
| M5        | 1.0     | GA                                              | 2–3 weeks     |
| M6        | 1.x     | Engine stages 2–4                               | multi-release |

Calendar estimate to 1.0 with the staffing above: about 4–5 months.

---

## M0. Environment (0.5 day)

| ID   | Work                                                                                                                                                                                                                                                          | Done when                                                               |
| ---- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| M0.1 | Rebase the RFC branch on `upstream/main`. Upstream has 4 newer commits, one of which (`c33ab96b6`) changes `rust/src/telemetry.rs`.                                                                                                                           | `git rev-list --count HEAD..upstream/main` prints `0`                   |
| M0.2 | Local prerequisites: Valkey 9.2 server, Docker, toolchains `1.94.1` (current CI) and `1.88` (target MSRV), `cargo-deny`, `cargo-hack`, `cargo-semver-checks`, and `cargo-public-api` (installed with stable; needs a nightly toolchain installed, not active) | `valkey-server --version` shows 9.2; `rustup toolchain list` shows both |
| M0.3 | Post the RFC and the Path Hash issue upstream (drafts exist), link them from #6915                                                                                                                                                                            | Issues exist and are linked                                             |

## M1. Guardrails (0.10.x)

No public API change. Each item is a CI or metadata PR.

| ID   | Work                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Files                                                                               | Tracking     | Depends on | Done when                                                                                                                                                                                                                                                                                                    | Est. |
| ---- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------- | ------------ | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---- |
| M1.1 | **L1 gate:** fail CI if a package named `redis` (crates.io redis-rs) enters the graph                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | `.github/workflows/rust.yml`                                                        | RFC L1       | —          | Step runs `cargo tree -e normal -i redis --target all` for default, `--no-default-features` and `--all-features`, and passes only when the output contains `did not match any packages`. Verified locally: all three print that and exit 101. Checking the exit code alone would also pass on network errors | 0.5  |
| M1.2 | **L2 gate:** commit a public API snapshot and fail on any path containing `redis::` or `glide_core_engine`, and on an unreviewed diff                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | `rust/public-api.txt`, `rust.yml`                                                   | RFC L2       | —          | `cargo public-api -sss` (run in `rust/`) or the `public-api` snapshot test from the tool's README matches the committed snapshot. Confirm the feature flags it accepts with `cargo public-api --help` and snapshot with all features enabled                                                                 | 1    |
| M1.3 | **Feature matrix:** build and clippy for every feature combination; CI never builds `--no-default-features` today                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | `rust.yml`                                                                          | RFC CI gates | —          | `cargo hack clippy --feature-powerset --all-targets -- -D warnings` passes                                                                                                                                                                                                                                   | 0.5  |
| M1.4 | **MSRV 1.88.** (a) `rust.yml`'s test job takes its toolchain from `rust-version` (the `msrv` step reads `rust/Cargo.toml`), and the committed `rust/Cargo.lock` pins `aws-config` 1.13.0, which needs Rust 1.94.1. So first move the test job to an explicit toolchain pin (keep 1.94.1) instead of `rust-version`. (b) Then set `rust-version = "1.88"` on `valkey-glide`, `glide-core`, `glide-telemetry`, `glide-logger`. (c) Add an MSRV job: install 1.88, regenerate the lockfile with `CARGO_RESOLVER_INCOMPATIBLE_RUST_VERSIONS=fallback cargo generate-lockfile` (not `--locked`), then build, clippy, unit and integration tests with `--all-features`. (d) Update the MSRV text in `rust/AGENTS.md` and `rust/docs/redis-rs-compatibility.md` | 4 `Cargo.toml`, `rust.yml`, `rust/AGENTS.md`, `rust/docs/redis-rs-compatibility.md` | #7175, #6959 | M1.3       | Test job still green on 1.94.1 with `--locked`; MSRV job green on 1.88. Already verified locally for `valkey-glide` on 1.88 with a fallback lockfile: 320 unit and 1,221 integration tests pass                                                                                                              | 1.5  |
| M1.5 | **docs.rs build** with `--cfg docsrs`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | `rust.yml`                                                                          | #7297        | —          | Job builds docs as docs.rs does                                                                                                                                                                                                                                                                              | 0.5  |
| M1.6 | Makefile targets (`rust`, `rust-test`, `rust-lint`) and `rust/target` caching                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | `Makefile`, `rust.yml`                                                              | #6865, #6869 | —          | `make rust-lint` runs the gate                                                                                                                                                                                                                                                                               | 1    |
| M1.7 | Retag Redis-only entries in `differences.json` (vector sets, `BITOP DIFF/DIFF1/ANDOR/ONE`, `digest`, `xcfgset`, `xnack`, `xinfo_stream_with_idempotency`) with a Redis-only reason instead of `TODO #7060`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | `src/parity_tests/differences.json`                                                 | #7060        | —          | `cargo unit-tests` passes; #7060's remaining list is only Valkey-applicable methods                                                                                                                                                                                                                          | 0.5  |
| M1.8 | Add Rust to the cross-language trackers #6292 and #7299                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | GitHub                                                                              | —            | —          | Rust column present                                                                                                                                                                                                                                                                                          | 0.5  |

## M2. Breaking batch (0.11)

Every break lands here so 1.0 has no known-wrong semantics. Order matters:
naming and layout first, so later items don't churn twice.

### M2a. API shape (do first, in this order)

| ID   | Work                                                                                                                                                                                  | Tracking              | Depends on | Done when                                                      | Est. |
| ---- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------- | ---------- | -------------------------------------------------------------- | ---- |
| M2.1 | `with_*` builder naming                                                                                                                                                               | #7702 (PR #7723 open) | —          | PR merged; `migration.md` updated                              | 1    |
| M2.2 | Option types in one private `src/options/` module, exported at the crate root only                                                                                                    | #7385                 | M2.1       | One public path per option type (checked by the M1.2 snapshot) | 2    |
| M2.3 | Routing types shaped like redis-rs, GLIDE-owned, Valkey terms (`Primary`), validated `Slot` newtype rejecting ≥ 16384                                                                 | #7678                 | M2.2       | `Route::SlotId` with 20000 cannot be constructed               | 2    |
| M2.4 | Public `Cmd::<name>` constructors and `Pipeline::add_command`                                                                                                                         | #7292                 | M2.2       | `Cmd::get("k").query_async(&c)` compiles and passes a test     | 1    |
| M2.5 | redis-rs **2.0** parity target, then **freeze** the snapshot: bump `REDIS_RS_VERSION` in `src/parity_tests/mod.rs` from `1.7.0`, adopt 2.0's typed-return fixes or record differences | #7641                 | M2.2       | Guard green against 2.0; `migration.md` covers 1.x and 2.x     | 4    |

### M2b. Behavior fixes (parallel after M2a)

| ID    | Work                                                                                                                                                                                                 | Tracking         | Done when                                                                                               | Est. |
| ----- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------- | ------------------------------------------------------------------------------------------------------- | ---- |
| M2.6  | Invalid config maps to `GlideError::Configuration`                                                                                                                                                   | #7665 (PR #7715) | merged                                                                                                  | 0.5  |
| M2.7  | `WATCH`-aborted `exec` is distinguishable from an empty batch                                                                                                                                        | #7672 (PR #7714) | merged                                                                                                  | 0.5  |
| M2.8  | Reject multi-node routes for batches (core + Rust)                                                                                                                                                   | #7673 (PR #7677) | merged; all bindings' CI green                                                                          | 1    |
| M2.9  | Lazy-connect failure returns `GlideError::Connection` (core + Rust)                                                                                                                                  | #7653 (PR #7685) | merged                                                                                                  | 0.5  |
| M2.10 | Subscribe/unsubscribe wait for server confirmation                                                                                                                                                   | #7686            | Publish immediately after `subscribe` is received in a test                                             | 2    |
| M2.11 | `pubsub_numsub`/`pubsub_shardnumsub` return `HashMap<Bytes, i64>`, like the other clients                                                                                                            | #7688            | Return type changed; tests updated                                                                      | 1    |
| M2.12 | Sharded pub/sub cluster-only, modeled on the C# client (separate subscription builders, cluster-only trait for sharded commands)                                                                     | #7691            | Sharded subscribe not available on `GlideClient`                                                        | 2    |
| M2.13 | Reject pub/sub with RESP2 (core + Rust)                                                                                                                                                              | #7683            | `GlideError::Configuration` saying Pub/Sub requires RESP3, when connecting with subscriptions and RESP2 | 1    |
| M2.14 | Typed multi-node server commands work on cluster clients and return each node's reply (as Java's `ClusterValue`, Python/Node dictionaries)                                                           | #7362            | Cluster integration tests for each affected method                                                      | 2    |
| M2.15 | Stop keyless `scan`/`scan_match` on cluster clients from paging random nodes: a standalone-only trait or an error pointing to `cluster_scan` (option chosen in #6872); standalone behavior unchanged | #6872            | Chosen behavior tested on cluster; doc comments and `TODO` markers updated; parity guard passes         | 2    |
| M2.16 | OpenTelemetry exporters configured with endpoint strings                                                                                                                                             | #7701            | merged                                                                                                  | 1    |
| M2.17 | Deprecated `ZRANGE` variants                                                                                                                                                                         | #7633 (PR #7647) | merged                                                                                                  | 0.5  |
| M2.18 | Reject empty/whitespace AZ in `ReadFrom`                                                                                                                                                             | #7815            | Config error test                                                                                       | 0.5  |
| M2.19 | Triage "Connection in recovery" retryability                                                                                                                                                         | #7169            | Decision recorded; fix if confirmed                                                                     | 0.5  |

### M2c. New API that changes defaults or adds lifecycle (after M2a)

| ID    | Work                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Tracking                           | Done when                                                                                                                                                                    | Est.                 |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------- |
| M2.20 | **Explicit `close()`**: `async fn close(&self)`, `fn is_closed(&self) -> bool` on both clients; blocking `close` on sync clients. Needs a `glide-core` close entry point (closed flag in `ClientShared`, checked on dispatch; drop the inner client so connections close)                                                                                                                                                                                                                                                                         | new issue (RFC "Client lifecycle") | After `close`, every clone returns `GlideError::Closing`; second `close` returns immediately; dropping without `close` still frees everything (test with a connection count) | 4                    |
| M2.21 | **Blocking client inside a Tokio runtime** returns `GlideError::Configuration` instead of panicking (`Handle::try_current()` check)                                                                                                                                                                                                                                                                                                                                                                                                               | new issue (RFC "Blocking layer")   | Test calling a sync method inside `#[tokio::test]` gets the error                                                                                                            | 0.5                  |
| M2.22 | `from_url` accepts `valkey://`, `valkeys://`, `?protocol=`; GLIDE-owned parser                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | #7031                              | URL tests for each scheme                                                                                                                                                    | 2                    |
| M2.23 | **Cargo features in `glide-core`/`glide-telemetry`** (`tls`, `tls-aws-lc`, `tls-ring`, `iam`, `telemetry`, `compression`), default-on; bindings enable all. Re-export as `valkey-glide` features with `#[doc(cfg)]`                                                                                                                                                                                                                                                                                                                               | RFC "Cargo features"               | `cargo hack --feature-powerset` green; every binding's CI green; crate count for `--no-default-features` recorded                                                            | 5 (glide-core owner) |
| M2.24 | **Engine stage 0:** delete fork code with no consumer, confirmed by building every crate after each removal. Candidates: `sentinel.rs`, `aio/connection_manager.rs` (the `connection-manager` feature is still enabled in `glide-core/Cargo.toml`, `java/Cargo.toml` and `python/glide-async/Cargo.toml` and must be removed there; check `tokio-retry2`'s other uses), `cluster_pipeline.rs`, and the `dummy-for-ort` package in `glide-core/redis-rs/Cargo.toml`. Then rename the fork's lib from `redis` to `glide_core_engine` in every crate | #2710                              | Every binding's CI green; fork line count and release binary size recorded before/after                                                                                      | 5 (glide-core owner) |

**M2 exit:** release `valkey-glide` 0.11.0 through `rust-cd.yml`, which
publishes, in order: `glide-logger`, `glide-telemetry`,
`glide-libfabric-sys` (`glide-rdma/glide-libfabric-sys`), `glide-rdma`,
`glide-core-engine`, `glide-core`, `valkey-glide` (`rust/`). Bump each
changed crate's version, and the `version` next to each `path` dependency. The CHANGELOG lists every break
with a one-line migration.

## M3. Command coverage (0.12+)

Order follows `command-gap-analysis.md` section 7. Each row is one PR.

### M3.0. Module test infrastructure (first)

| ID    | Work                                                                                                                                                                                       | Done when                                                     | Est. |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------- | ---- |
| M3.0a | Harness accepts external endpoints (for example `GLIDE_STANDALONE_ENDPOINTS`, `GLIDE_CLUSTER_ENDPOINTS`) for a separate `tests/modules_*.rs` target that the `it_*` alias does not pick up | `cargo test --test 'modules_*'` runs against a given endpoint | 2    |
| M3.0b | New `.github/workflows/rust-modules.yml`, modeled on `node-modules.yml`: `./.github/actions/start-valkey-docker` with `image: valkey/valkey-bundle:9.2.0`                                  | Workflow green on a smoke test                                | 1    |

### M3.1. Valkey 9.x data commands

| ID   | Commands                                         | Tracking                             | Est. |
| ---- | ------------------------------------------------ | ------------------------------------ | ---- |
| M3.1 | `DELEX`                                          | #7238 (PR #7843 open, includes Rust) | 0.5  |
| M3.2 | `INCREX`                                         | #7200                                | 1    |
| M3.3 | `MSETEX`                                         | #6152                                | 1    |
| M3.4 | `XACKDEL`, `XDELEX` (`KEEPREF`/`DELREF`/`ACKED`) | #7197                                | 1    |
| M3.5 | `HSETEX` `NX`/`XX`                               | #7873 (PR #7874 open)                | 0.5  |
| M3.6 | `EVAL_RO`, `EVALSHA_RO` with `ReadFrom` routing  | #7880                                | 2    |
| M3.7 | `CONFIG INFO`                                    | new issue                            | 0.5  |

### M3.2. Modules

| ID    | Work                                                                                                                                                                                                                                                                                                                | Tracking            | Depends on  | Est.  |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------- | ----------- | ----- |
| M3.8  | `BloomCommands`: 9 `BF.*`; `bf_scandump`/`bf_loadchunk` recorded as differences (valkey-bloom uses `BF.LOAD`)                                                                                                                                                                                                       | #4649               | M3.0        | 3–4   |
| M3.9  | `JSON.MERGE`; `JSON.MSET` (multi-key, check core multi-slot handling first)                                                                                                                                                                                                                                         | #5883               | M3.0        | 1–2   |
| M3.10 | **Core search fixes** (glide-core owner): #7593 routing, #7859 read-only list, #7884 `WITHCURSOR` conversion                                                                                                                                                                                                        | #7593, #7859, #7884 | —           | 3     |
| M3.11 | `SearchCommands` for valkey-search 1.3: `FT.CREATE`, `FT.SEARCH`, `FT.AGGREGATE`, `FT.HYBRID`, `FT.CURSOR READ/DEL`, `FT.INFO`, `FT.DROPINDEX`, `FT._LIST`. Retarget #7286 from redis-rs's RediSearch shape. Do **not** add `FT.ALIAS*`/`FT.EXPLAIN*`/`FT.PROFILE` (not in valkey-search 1.3). Cursor ids are `u64` | #7286               | M3.0, M3.10 | 10–15 |

### M3.3. Path Hash

| ID    | Work                                                                                                                                                                                                             | Tracking                | Depends on | Est. |
| ----- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------- | ---------- | ---- |
| M3.12 | Core: add read `PH*` commands to `is_readonly_cmd` (`glide-core/redis-rs/redis/src/cluster_routing.rs`); normalize `PHGETALL` and `PHPREFIXES WITHVALUES` to maps in `glide-core/src/client/value_conversion.rs` | Path Hash issue (draft) | —          | 1    |
| M3.13 | `PathHashCommands` behind opt-in `unstable-pathhash`; byte cursor iterator for `PHPATHSCAN` (`ScanIter` is `u64`-only)                                                                                           | Path Hash issue         | M3.12      | 3–4  |

### M3.4. Management

| ID    | Commands                                                                                                                                                         | Tracking                   | Est. |
| ----- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------- | ---- |
| M3.14 | Read-only `CLUSTER` (`INFO`, `NODES`, `SHARDS`, `SLOTS`, `MYID`, `MYSHARDID`, `KEYSLOT`, `COUNTKEYSINSLOT`, `GETKEYSINSLOT`, `LINKS`, `REPLICAS`, `SLOT-STATS`)  | #7299                      | 3    |
| M3.15 | `COMMANDLOG`, `SLOWLOG`, `HOTKEYS` (9.2), `MODULE LIST`, `CLIENT KILL/LIST/INFO`                                                                                 | #6292, #7196               | 3    |
| M3.16 | `ACL` (17 redis-rs methods) plus 9.2 roles (`SETROLE`, `GETROLE`, `DELROLE`, `ROLES`, `DIGEST`)                                                                  | #7060, new issue for roles | 3    |
| M3.17 | `CLUSTER` admin and atomic slot migration (`MIGRATESLOTS`, `GETSLOTMIGRATIONS`, `CANCELSLOTMIGRATIONS`, `FLUSHSLOT`)                                             | #7299, #6151               | 3    |
| M3.18 | `COMMAND *`, `MEMORY *`, `LATENCY HISTOGRAM`, `ROLE`, `SWAPDB`, `WAITAOF`, `SHUTDOWN`, `GEORADIUS*`                                                              | #6292, #7060               | 3    |
| M3.19 | Remaining #7060 methods (stream conveniences, `scan_options`, `zadd_options`, `invoke_script`, `load_script`) and #7082 option variants, `incr_by_float` (#7262) | #7060, #7082, #7262        | 4    |

**M3 exit:** every user-facing command in `command-gap-analysis.md` is
typed or explicitly excluded there; `differences.json` has no
`TODO #7060` entries left.

## M4. Feature parity and engine stage 1 (0.12+, parallel with M3)

| ID    | Work                                                                                                                                                                                                                                                                                                                                                     | Tracking                   | Depends on     | Est. |
| ----- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------- | -------------- | ---- |
| M4.1  | Circuit breaker configuration                                                                                                                                                                                                                                                                                                                            | #6916                      | —              | 2    |
| M4.2  | Client-side caching                                                                                                                                                                                                                                                                                                                                      | #6918                      | —              | 4    |
| M4.3  | Client info tag                                                                                                                                                                                                                                                                                                                                          | #6920                      | —              | 1    |
| M4.4  | Custom credentials provider                                                                                                                                                                                                                                                                                                                              | #6921                      | —              | 2    |
| M4.5  | Custom logger                                                                                                                                                                                                                                                                                                                                            | #6922                      | —              | 2    |
| M4.6  | Value compression                                                                                                                                                                                                                                                                                                                                        | #6878                      | M2.23          | 2    |
| M4.7  | OpenTelemetry command spans                                                                                                                                                                                                                                                                                                                              | #7699                      | —              | 2    |
| M4.8  | Unix socket connections (standalone)                                                                                                                                                                                                                                                                                                                     | #7645 (draft PR #7451)     | —              | 1    |
| M4.9  | `BigInt`/`BigUint` (`num-bigint` 0.5 per PR #7181) and optional `rust_decimal`/`bigdecimal` features                                                                                                                                                                                                                                                     | #7664, #7663               | M2.23          | 2    |
| M4.10 | Dedicated connections / pooling, following RFC #5815; then `MONITOR`                                                                                                                                                                                                                                                                                     | #6917, #5815, #6919        | #5815 accepted | 5    |
| M4.11 | `test-util` feature: mock executor with scripted replies, argument assertion helpers                                                                                                                                                                                                                                                                     | RFC "Testing support"      | —              | 3    |
| M4.12 | Cargo workspace for the published crates; `cargo publish --workspace` in `rust-cd.yml`. #7261 lists five crates, but `rust-cd.yml` also publishes `glide-libfabric-sys` and `glide-rdma`; decide whether they join                                                                                                                                       | #7261                      | M2.24          | 3    |
| M4.13 | **Engine stage 1** (glide-core owner): `glide_core::protocol` facade (`Value`, `Cmd`, `Pipeline`, `PushInfo`, `PushKind`, errors, routing types); remove `glide-core-engine` from the `Cargo.toml` of `ffi`, `java`, `node/rust-client`, `python/glide-async`, `rust`, `benchmarks/rust`; CI fails if any crate other than `glide-core` names the engine | RFC "Engine independence"  | M2.24          | 8–10 |
| M4.14 | Extend the parity guard beyond the command table (hotkeys, pub/sub, `macros.rs` methods)                                                                                                                                                                                                                                                                 | #7288                      | M2.5           | 2    |
| M4.15 | Test coverage for untested extension-trait commands; migrate tests to the shared matrix                                                                                                                                                                                                                                                                  | #7236, #7289, #6996, #6997 | —              | 5    |

## M5. GA (1.0)

| ID   | Work                                                                                                                                                                                                                                                                                         | Tracking           | Done when                                        | Est. |
| ---- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------ | ------------------------------------------------ | ---- |
| M5.1 | Benchmarks: add `valkey-glide` to `benchmarks/rust` (it benchmarks `glide-core` directly today) and compare with redis-rs `MultiplexedConnection`/`cluster_async`: GET/SET 100 B and 16 KB, 90/10, pipelines of 100, cluster MGET over 3 and 9 shards; p50/p99, throughput, RSS, binary size | RFC "Benchmarking" | Report published; no regression against redis-rs | 4    |
| M5.2 | Docs refresh: README, `DESIGN.md`, `migration.md` (redis-rs 1.x and 2.x), feature-flag table, "Valkey-only features" page                                                                                                                                                                    | #7205, #7295       | Docs reviewed                                    | 3    |
| M5.3 | glide.valkey.io: Rust quickstart and Rust tabs in the concept pages (the docs repo has only `concepts/architecture/rust-core-design.mdx` today)                                                                                                                                              | valkey-glide-docs  | Rust selectable on the quickstart                | 3    |
| M5.4 | `examples/rust/` (does not exist yet): standalone, cluster, pipeline, pub/sub, IAM, search, bloom                                                                                                                                                                                            | —                  | Examples build in CI                             | 2    |
| M5.5 | Release candidate, bug bash, then `cargo semver-checks` gate against 1.0 from then on                                                                                                                                                                                                        | #6915              | 1.0.0 published; #6915 closed                    | 3    |

**M5 entry criteria:** M2 shipped; M3.1–M3.13 done; M4.1–M4.9, M4.11 and
M4.13 done; all `breaking` + `Rust` issues closed.
M3.14–M3.19 (management commands) are additive and may land in 1.x minors
after GA.

## M6. Engine independence, stages 2–4 (1.x, glide-core owner)

Each stage ships behind a compile-time `engine-next` feature in `glide-core`,
with every binding's suite run on both implementations; the default flips in
one release and the old code is deleted in the next.

| Stage | Work                                                                                                                                             | Exit                                                                                                  |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------- |
| 2     | RESP2/RESP3 parser, encoder, `Value`, `Cmd`, errors; fuzz targets, `proptest` round trips, differential tests against the old parser             | No benchmark regression; fuzzing clean for an agreed run time                                         |
| 3     | Multiplexed connection, handshake, TLS, push handling; move GLIDE-authored modules (cache, retries, buffer pool, pub/sub synchronizer) unchanged | Standalone suites of every binding pass on both                                                       |
| 4a    | Move cluster modules into `glide-core` unchanged; delete `glide-core-engine`                                                                     | Every binding's suite passes with no glue changes; PHP picks it up on its next submodule bump         |
| 4b    | Refactor cluster sub-parts in place (slot map and topology first, `cluster_async/mod.rs` last), each with a differential test                    | Cluster, failover and slot-migration suites pass on both, including PHP's (needs a cross-repo CI job) |

## Dependency graph (critical path)

```text
M0 ─► M1 ─► M2a (M2.1 → M2.2 → {M2.3, M2.4, M2.5}) ─► M2b, M2c ─► 0.11
                                   │
M2.23 (features) ─► M4.6, M4.9     M2.5 ─► M3.* (new methods follow 2.0 shapes)
M2.24 (stage 0)  ─► M4.12, M4.13 ─► M6
M3.0 (module CI) ─► M3.8, M3.9, M3.11
M3.10 (core FT)  ─► M3.11
M3.12 (core PH)  ─► M3.13
M2 + M3.1–M3.13 + M4 (listed) ─► M5 ─► 1.0
```

Critical path to 1.0: M1.4 → M2.1 → M2.2 → M2.5 → M3.11 (search, the largest
item) → M5.

## Risks

| Risk                                                                                                    | Mitigation                                                                                                        |
| ------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| `glide-core` changes (M2.8, M2.9, M2.13, M2.20, M2.23, M2.24, M3.10, M3.12, M4.13) break other bindings | `glide-core` reviewer on every such PR; all binding CI must pass; land them early in their milestone              |
| redis-rs 2.0 is still a release candidate (2.0.0-rc.4)                                                  | Do M2.5 against 2.0.0 final; if 2.0.0 is not out when M2a starts, pin the latest RC and re-run the guard at final |
| Path Hash API changes upstream (valkey#4706)                                                            | `unstable-pathhash` feature; `#[non_exhaustive]` options                                                          |
| MSRV 1.88 depends on fallback resolution picking older AWS SDK releases                                 | MSRV CI job (M1.4); raise `rust-version` only in a minor release                                                  |
| Module CI image drift                                                                                   | Pin `valkey/valkey-bundle:9.2.0` by tag, as `start-valkey-docker` already pins 9.1.3                              |
| Search scope is large (FT.CREATE/SEARCH/AGGREGATE/HYBRID options)                                       | Split M3.11 into one PR per command; follow the shapes Java landed in #7379                                       |
| Breaking batch slips                                                                                    | M2 has no new commands; M3 can start against the M2a API once M2.2 and M2.5 merge                                 |

## Decisions already made

Recorded from the RFC review:

- `iam`, `telemetry`, `compression` stay default-on at 1.0.
- The redis-rs parity snapshot freezes at 2.0.
- No compat crate for redis-rs-based libraries.
- Explicit `close()` as specified in M2.20.
- MSRV 1.88 for the published crates.
- Engine independence is part of the plan and owned by `glide-core`; stage 4
  is move-then-refactor.

## Open decisions

1. Whether a GLIDE-supported server implements `FT.ALIAS*`/`FT.EXPLAIN*`/
   `FT.PROFILE`; until confirmed, Rust omits them (M3.11).
2. Whether `SUBSTR` and `HMSET` get typed APIs (low value; M3.18 can skip).
3. Release cadence between 0.11 and 1.0 (one minor per milestone, or per
   few weeks).
