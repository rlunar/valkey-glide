# Agent task list: full Valkey client for Rust

Hand-off list for AI coding agents, with human verification by
**@madolson** and **@rlunar**. Every task is test-first (TDD). Derived from
[implementation-plan.md](implementation-plan.md); task IDs keep the plan's
milestone numbers so the two documents line up.

Facts in the task specs (syntax, replies, RESP2/RESP3 differences) were
checked against Valkey 9.2.0 and `valkey/valkey-bundle:9.2.0` (valkey-json
1.1.0, valkey-bloom 1.0.1, valkey-search 1.3.0). redis-rs signatures come
from `rust/src/parity_tests/redis_parity.json` (redis-rs 1.7.0).

## 1. How a task flows

1. **Pick** a task whose dependencies are merged. One agent per task.
2. **Branch** from the integration branch `rust/next` on `rlunar/valkey-glide`
   (created in H0.3), named `agent/<task-id>-<slug>`.
3. **Red commit:** tests only. Run them and capture the failing output. A
   compile error because the method does not exist yet counts as red. Commit
   message: `test(rust): <what> (red)`.
4. **Green commit:** the smallest implementation that passes. Commit message
   in Conventional Commits, for example `feat(rust): add INCREX`.
5. **Refactor commit** (optional): cleanup with tests still green.
6. **Gate:** run the full quality gate (section 3) and paste the summary.
7. **Open a PR** against `rust/next` on the fork with the template in
   section 4. Request the reviewers listed on the task.
8. **Humans verify** (section 5), then squash-merge into `rust/next`.
   Upstream PRs to `valkey-io/valkey-glide` are opened by a human from
   `rust/next`, grouped by milestone.

Keep red and green as separate commits so a reviewer can check out the red
commit and see the failure.

## 2. Standing instructions for every agent

Paste this block at the top of every agent prompt.

```text
You are implementing one task in the valkey-glide monorepo (Rust client in rust/).

Before writing code, read: AGENTS.md, CLAUDE.md, rust/AGENTS.md,
rust/DEVELOPER.md, rust/docs/dev/options.md, and the files your task lists.
If you touch glide-core, also read glide-core/AGENTS.md.

Process (test-driven):
1. Write the tests in the task's "Red" section first. Run them. Save the
   failing output. Commit tests only: "test(rust): <what> (red)".
2. Implement the minimum to make them pass. Commit with a Conventional Commit
   message, scope `rust` (or `core` for glide-core).
3. Run the full quality gate. Every step must pass.
4. Open a PR against rust/next on rlunar/valkey-glide using the PR template.

Rules:
- Do not weaken, skip, or delete an existing test to make yours pass.
- New integration tests must run, not skip, on Valkey 9.2. Paste the lines
  showing they ran.
- Every commit uses `git commit -s` (DCO).
- No `unsafe` in rust/ (`#![forbid(unsafe_code)]`). Every public item needs a
  doc comment (`#![deny(missing_docs)]`).
- Follow rust/docs/dev/options.md for every option type.
- Where redis-rs has the method, match its name, generic order and argument
  list from rust/src/parity_tests/redis_parity.json. Use Valkey terminology
  (primary/replica) in GLIDE-owned types.
- Update src/parity_tests/differences.json when the command table changes.
- Add a CHANGELOG.md line under "## Pending ..." in the existing style.
- Touch only the files your task needs. No unrelated refactors or formatting.
- Do not push to valkey-io/valkey-glide. Do not publish crates.
- If the spec is ambiguous or contradicts the server's behavior, stop and
  write the question in the PR description instead of guessing.
```

## 3. Quality gate

Run in `rust/` unless the task says otherwise.

```bash
cargo fmt --all -- --check
cargo clippy --all-features --all-targets -- -D warnings
cargo clippy --all-targets -- -D warnings
cargo deny --config ../deny.toml check
RUSTDOCFLAGS="-Dwarnings" cargo doc --no-deps --document-private-items
cargo unit-tests
cargo doc-tests
cargo integration-tests
```

- Integration tests need `valkey-server` 9.2 on `PATH` (or
  `VALKEY_SERVER_PATH`), a `valkey-cli` on `PATH`, and `python3`.
- Module tests (after T1.8) also need
  `docker run --rm -p 127.0.0.1:6379:6379 valkey/valkey-bundle:9.2.0` and the
  endpoint variables from T1.8.
- Tasks that touch `glide-core` also run that crate's gate
  (`glide-core/AGENTS.md`). Every binding's CI must pass on the PR.
- A dependency change ships an updated `rust/Cargo.lock` (CI uses `--locked`).

## 4. PR template (agent fills in)

```markdown
## Task

<task id and title> — closes/relates <issue>

## Red

Commit: <sha>
<failing test output, trimmed to the relevant lines>

## Green

Commit: <sha>
<passing test output for the new tests, showing they ran on Valkey 9.2 (not SKIP)>

## Gate

<one line per gate step: pass/fail>

## Behavior notes

<anything the server did that differed from the task spec, and how you handled it>

## Public API changes

<new or changed public items, copied from the docs>

## Questions for reviewers

<or "none">
```

## 5. Human verification

| Reviewer      | Verifies                                                                                                                                                                       |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **@madolson** | Server semantics match Valkey (arguments, replies, errors, edge cases), cluster routing and read-only flags, RESP2/RESP3 normalization, every `glide-core` change, engine work |
| **@rlunar**   | Rust API shape and naming, redis-rs parity, docs and doc comments, CI and release, CHANGELOG                                                                                   |
| **Both**      | Breaking changes, `glide-core` changes that affect other bindings, anything marked **design sign-off**                                                                         |

Reviewer checklist for every PR:

1. Check out the red commit, run the new tests, see them fail for the stated
   reason (not an unrelated compile error elsewhere).
2. Check out the green commit, see them pass. Confirm they ran on 9.2.
3. The tests assert the behavior in the task spec, including the listed edge
   cases. A test that only checks "no error" is not enough.
4. The implementation does not special-case the test.
5. Gate output is complete. CI is green.
6. Public API matches the task's signature, or the PR explains why not.

Task types:

- **agent** — ready to hand over.
- **agent + design sign-off** — the agent first posts a short design (public
  signatures, behavior, files) as a PR comment; implementation starts after
  both reviewers approve it.
- **human** — decisions, coordination, releases.

## 6. Tasks

Tasks inside a wave can run in parallel unless a dependency says otherwise.
To avoid merge conflicts, run at most one task at a time per command family
file (`src/commands/<family>.rs`) and per `glide-core` file.

### Wave 0 — setup (human)

| ID   | Task                                                                                                                                             | Owner   |
| ---- | ------------------------------------------------------------------------------------------------------------------------------------------------ | ------- |
| H0.1 | Rebase `rfc/rust-without-redis-rs` on `upstream/main` (4 commits behind, one touches `rust/src/telemetry.rs`)                                    | @rlunar |
| H0.2 | Post the RFC and the Path Hash issue upstream; link them from #6915                                                                              | @rlunar |
| H0.3 | Create `rust/next` on `rlunar/valkey-glide` from current `upstream/main`; enable CI on the fork                                                  | @rlunar |
| H0.4 | Coordinate with authors of open PRs this list depends on (#7715, #7714, #7677, #7723, #7647, #7685, #7843, #7874) so agents don't duplicate them | @rlunar |
| H0.5 | Decide the redis-rs 2.0 target: final release, or latest RC if final isn't out (blocks T2.5)                                                     | both    |

### Wave 1 — guardrails (no public API change)

**T1.1 L1 gate: no crates.io `redis` in the graph** — agent — reviewers: @rlunar

- Files: new `rust/scripts/check-no-redis-rs.sh`, `.github/workflows/rust.yml`.
- Spec: for default, `--no-default-features` and `--all-features`, run
  `cargo tree -e normal -i redis --target all`. Pass only when the output
  contains `did not match any packages`. Any other output or error fails
  (so a network error doesn't pass).
- Red: run the script with an argument pointing at a scratch crate whose
  `Cargo.toml` depends on `redis = "1"` and on `valkey-glide` by path; it
  must exit non-zero. Run it on `rust/`; it must exit 0. Add both as a CI
  self-test step.
- Done when: CI job runs the gate and the self-test.

**T1.2 L2 gate: public API snapshot** — agent — reviewers: @rlunar

- Files: `rust/` snapshot file, `rust.yml`.
- Spec: `cargo public-api` (install with stable; needs a nightly toolchain
  installed). Confirm accepted feature flags with `cargo public-api --help`;
  snapshot with all features enabled. CI fails on a diff and on any line
  containing `redis::` or `glide_core_engine`.
- Red: add a temporary `pub fn leak() -> redis::Value` in a scratch commit
  and show the check fails; remove it.
- Done when: snapshot committed; CI check green on `rust/next`.

**T1.3 Feature-powerset build** — agent — reviewers: @rlunar

- Spec: `cargo hack clippy --feature-powerset --all-targets -- -D warnings`
  in CI.
- Red: show `cargo clippy --no-default-features --all-targets -- -D warnings`
  output today (it may already warn; CI never runs it). Fix any warnings
  found.
- Done when: CI step green.

**T1.4 MSRV 1.88** — agent — reviewers: @rlunar, @madolson (touches
`glide-core`, `glide-telemetry`, `glide-logger` manifests)

- Spec, in order:
    1. `rust.yml`'s `msrv` step reads `rust-version` and installs it for the
       test job, and `rust/Cargo.lock` pins `aws-config` 1.13.0, which needs
       Rust 1.94.1. Change the test job to an explicit toolchain pin (1.94.1)
       first.
    2. Set `rust-version = "1.88"` in `rust/`, `glide-core/`,
       `glide-telemetry/`, `glide-logger/`.
    3. New MSRV job: install 1.88;
       `CARGO_RESOLVER_INCOMPATIBLE_RUST_VERSIONS=fallback cargo generate-lockfile`
       (no `--locked`); then build, clippy, `cargo unit-tests`,
       `cargo integration-tests`, all with `--all-features`.
    4. Update MSRV text in `rust/AGENTS.md` and
       `rust/docs/redis-rs-compatibility.md`.
- Red: show that `cargo +1.88 check --locked --lib` fails on the committed
  lockfile. Verified error: `rustc 1.88.0 is not supported by the following
packages: aws-config@1.13.0 requires rustc 1.94.1` (same with
  `rust-version` at 1.94.1 or 1.88). This is why step 1 comes first.
- Done when: both jobs green. Known local result for reference: on 1.88 with
  a fallback lockfile, 320 unit and 1,221 integration tests pass.
- Relates: #7175, #6959.

**T1.5 docs.rs build** — agent — reviewers: @rlunar — #7297

- Red: a CI step building docs with `RUSTDOCFLAGS="--cfg docsrs"` on
  nightly, as docs.rs does; show current result.
- Done when: step green.

**T1.6 Makefile targets and cache** — agent — reviewers: @rlunar — #6865, #6869

- Spec: `make rust`, `make rust-test`, `make rust-lint` (the section 3 gate).
- Red: `make rust-lint` fails with "No rule to make target".
- Done when: targets run the gate; `make help` lists them.

**T1.7 Retag Redis-only parity entries** — agent — reviewers: @rlunar — #7060

- Spec: in `src/parity_tests/differences.json`, change the reason from
  `TODO #7060: not implemented yet` to a Redis-only reason for: the 17
  vector-set methods (`vadd` … `vsim_options`), `bit_and_or`, `bit_diff`,
  `bit_diff1`, `bit_one`, `digest` (already Redis-only), `xcfgset`, `xnack`,
  `xinfo_stream_with_idempotency`. Valkey 9.2 has no `VADD`, `XNACK` or
  `XCFGSET`, and its `BITOP` accepts only `AND`, `OR`, `XOR`, `NOT`.
- Red: add a unit test asserting no entry with a `TODO #7060` reason names a
  method in that list.
- Done when: `cargo unit-tests` green.

**T1.8 Module test infrastructure** — agent — reviewers: @rlunar, @madolson

- Spec: the harness (`tests/common/server.rs`, `tests/common/cluster.rs`)
  only starts its own servers. Add a separate test target family
  `tests/modules_*.rs` (not matched by the `it_*` alias) that reads
  `GLIDE_STANDALONE_ENDPOINTS` and `GLIDE_CLUSTER_ENDPOINTS` and fails with a
  clear message when unset. New workflow `.github/workflows/rust-modules.yml`
  modeled on `node-modules.yml`, using `./.github/actions/start-valkey-docker`
  with `image: valkey/valkey-bundle:9.2.0` (the action defaults to 9.1.3).
- Red: a `modules_smoke.rs` test that runs `MODULE LIST` and asserts `json`,
  `bf` and `search` are loaded; it fails because the harness doesn't exist.
- Done when: `cargo test --test 'modules_*'` passes locally against the
  bundle container and in the new workflow.

### Wave 2 — API shape (sequential, breaking, 0.11)

Run in this order. Each needs **design sign-off** from both reviewers.

| ID   | Task                                                                                                                        | Tracking                                                    | Depends on | Red test                                                                                             |
| ---- | --------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------- | ---------- | ---------------------------------------------------------------------------------------------------- |
| T2.1 | `with_*` builder naming                                                                                                     | #7702, PR #7723 (finish or rebase that PR; don't duplicate) | H0.4       | Doctest in `migration.md` using new names fails to compile                                           |
| T2.2 | Option types in a private `src/options/`, exported at the crate root only                                                   | #7385                                                       | T2.1       | Public API snapshot (T1.2) test expecting no `glide::commands::options::` paths fails                |
| T2.3 | Routing types: GLIDE-owned, redis-rs-shaped, Valkey terms (`Primary`), `Slot` newtype rejecting ≥ 16384                     | #7678                                                       | T2.2       | Unit test: `Slot::new(16384)` returns `None`; `Slot::new(16383)` returns `Some`                      |
| T2.4 | Public `Cmd::<name>` constructors and `Pipeline::add_command`                                                               | #7292                                                       | T2.2       | Integration test `Cmd::get(k).query_async(&c)` and `pipe().add_command(Cmd::get(k))` fail to compile |
| T2.5 | redis-rs 2.0 parity target, then freeze the snapshot (`REDIS_RS_VERSION` in `src/parity_tests/mod.rs`, currently `"1.7.0"`) | #7641                                                       | T2.2, H0.5 | Parity guard fails after the version bump; green after adopting changes or recording differences     |

### Wave 3 — fixes and lifecycle (0.11)

Bug fixes are regression-test-first: the red test reproduces the issue's
"Current behavior".

| ID    | Task                                                                                                                                       | Type                    | Reviewers | Red test (from the issue)                                                                                                                                                 |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| T3.1  | #7686 subscribe/unsubscribe wait for server confirmation                                                                                   | agent                   | both      | `subscribe` then immediate `publish` from a second client; assert receivers == 1, repeat 50 times                                                                         |
| T3.2  | #7688 `pubsub_numsub`/`pubsub_shardnumsub` return `HashMap<Bytes, i64>`                                                                    | agent                   | @rlunar   | Type-level test expecting a `HashMap` fails to compile                                                                                                                    |
| T3.3  | #7691 sharded pub/sub cluster-only (C#-style builders per the issue)                                                                       | agent + design sign-off | both      | Compile-fail doc test: `ssubscribe` on `GlideClient`                                                                                                                      |
| T3.4  | #7683 reject pub/sub with RESP2 (core + Rust)                                                                                              | agent                   | both      | Connect with subscriptions and RESP2; assert `GlideError::Configuration` mentioning RESP3 (today: timeout)                                                                |
| T3.5  | #7362 typed multi-node server commands on cluster return per-node replies                                                                  | agent + design sign-off | both      | Cluster test for each affected method; today fails with "Response type not string compatible"                                                                             |
| T3.6  | #6872 keyless `scan`/`scan_match` on cluster clients (standalone-only trait or error pointing to `cluster_scan`, per the issue's decision) | agent + design sign-off | both      | Cluster test showing the chosen behavior; standalone scan test unchanged                                                                                                  |
| T3.7  | #7701 OpenTelemetry exporters by endpoint string                                                                                           | agent                   | @rlunar   | Config test with an endpoint string                                                                                                                                       |
| T3.8  | #7815 reject empty/whitespace AZ in `ReadFrom`                                                                                             | agent                   | @rlunar   | Unit test: `""` and `"  "` give `GlideError::Configuration`                                                                                                               |
| T3.9  | #7169 "Connection in recovery" retryability                                                                                                | human triage first      | @madolson | —                                                                                                                                                                         |
| T3.10 | Open PRs #7715, #7714, #7677, #7685, #7647: review and finish with their authors                                                           | human                   | both      | —                                                                                                                                                                         |
| T3.11 | **Explicit `close()`** (spec below)                                                                                                        | agent + design sign-off | both      | see spec                                                                                                                                                                  |
| T3.12 | Blocking client inside a Tokio runtime returns `GlideError::Configuration`                                                                 | agent                   | @rlunar   | `#[tokio::test]` calling `SyncGlideClient::connect` (any address; it fails before connecting). Verified today: panics with `Cannot start a runtime from within a runtime` |
| T3.13 | `from_url` accepts `valkey://`, `valkeys://`, `?protocol=resp2\|resp3` with a GLIDE-owned parser                                           | agent                   | @rlunar   | URL parsing unit tests for each scheme and protocol value, plus invalid ones                                                                                              |

**T3.11 `close()` spec**

- API: `async fn close(&self)` and `fn is_closed(&self) -> bool` on
  `GlideClient` and `GlideClusterClient`; `fn close(&self)` on the sync
  clients.
- Core: a close entry point in `glide-core` (closed flag shared by all
  clones, checked on dispatch; drop the inner client so connections close).
- Red tests:
    1. After `close`, a command on another clone returns `GlideError::Closing`.
    2. A command in flight when `close` is called returns
       `GlideError::Closing` (use a `BLPOP` with a timeout on another clone).
    3. Second `close` returns immediately.
    4. `is_closed` is true on every clone after `close`.
    5. After `close`, the server's `CLIENT LIST` (from a separate client) no
       longer shows the closed client's connections.
    6. Dropping all clones without `close` also removes the connections.
- Not in scope: closing global OpenTelemetry exporters.

### Wave 4 — commands

Every command task has the same Red structure:

1. **Mock test** in `src/mock_tests/<family>.rs`: build the command through
   `Mock` and assert the exact arguments with `m.assert_args(&[…])`; return a
   scripted reply and assert the decoded value. Pattern:
   `let m = Mock::int(1); assert!(m.del_if_eq("k", "v").await.unwrap()); m.assert_args(&["DELIFEQ", "k", "v"]);`
2. **Option encoding test** with `assert_args` / `assert_args_empty` from
   `src/test_utils.rs`, covering every variant (per `options.md`).
3. **Integration test** in `tests/it_<family>.rs` with `matrix_test!` and
   `skip_if_version_below!(c, major, minor, patch)`, covering the edge cases
   listed in the task.

Use hash tags (`common::tkey`) for multi-key commands so cluster tests hit
one slot.

**T4.1 `DELEX`** — human coordinate first — #7238, PR #7843 (open, includes
Rust). Only hand to an agent if #7843 stalls.

- Syntax: `DELEX key (IFEQ | IFNE) value`. Since 9.2.0.
- Reply: integer `1` deleted, `0` not deleted (also `0` for a missing key).
- redis-rs: `del_ex<K: ToSingleRedisArg>(key: K, value_comparison: ValueComparison) -> usize`.

**T4.2 `INCREX`** — agent — reviewers: both (core change) — #7200

- Syntax:
  `INCREX key [NX | XX] [EX s | PX ms | EXAT ts | PXAT ts-ms | PERSIST] [ENX] [BYINT n | BYFLOAT f] [LBOUND lo] [UBOUND hi] [SATURATE]`.
  Since 9.2.0.
- redis-rs: `increx<K: ToSingleRedisArg, V: ToSingleRedisArg>(key: K, increment: V, options: IncrexOptions<V>) -> IncrexResult`.
- Verified behavior:
    - Reply is `[new_value, applied_increment]`.
    - No `BY…`: increments by 1. Missing key starts at 0.
    - `NX` on an existing key: no change, reply `[current, 0]`.
    - Out of bounds without `SATURATE`: no change, reply `[current, 0]`.
    - With `SATURATE`: clamps; for 10 with `BYINT 100 UBOUND 50 SATURATE`
      the reply is `[50, 40]`.
    - Without an expiry option, an existing TTL is kept.
    - `ENX` sets the expiry only when the key has none.
    - `PERSIST` removes the TTL.
    - Non-numeric value: error `ERR value is not an integer or out of range`.
    - **`BYFLOAT`: RESP2 replies with bulk strings, RESP3 with doubles.**
- Core: add an `expected_type_for_cmd` entry in
  `glide-core/src/client/value_conversion.rs` so `INCREX … BYFLOAT` returns
  doubles in both protocols (reference: line with
  `b"HINCRBYFLOAT" | b"INCRBYFLOAT" | b"ZINCRBY" => Some(ExpectedReturnType::Double)`).
  Integers stay integers.
- Red: core unit test converting the RESP2 bulk-string pair to doubles;
  Rust tests for each behavior above under RESP2 and RESP3.

**T4.3 `MSETEX`** — agent — reviewers: both — #6152

- Syntax:
  `MSETEX numkeys key value [key value …] [NX | XX] [EX s | PX ms | EXAT ts | PXAT ts-ms | KEEPTTL]`.
  Since 9.1.0.
- redis-rs: `mset_ex<K: ToRedisArgs, V: ToRedisArgs>(items: &[(K, V)], options: MSetOptions) -> bool`.
- Verified: reply integer `1` when set, `0` when `NX`/`XX` fails (`NX` with
  any existing key, `XX` with any missing key). All-or-nothing: on `0`, no
  key is written or changed. `PX` sets the TTL on every key.
- Cluster: multi-key; tests use hash tags. Note for @madolson: confirm
  whether glide-core splits `MSETEX` across slots or the server's CROSSSLOT
  error is expected.

**T4.4 `XACKDEL`, `XDELEX`** — agent — reviewers: both — #7197

- Syntax: `XACKDEL key group [KEEPREF | DELREF | ACKED] IDS numids id [id …]`;
  `XDELEX key [KEEPREF | DELREF | ACKED] IDS numids id [id …]`. Since 9.2.0.
- redis-rs: `xack_del(key, group, ids, options: StreamDeletionPolicy) -> Vec<XAckDelStatusCode>`;
  `xdel_ex(key, ids, options: StreamDeletionPolicy) -> Vec<XDelExStatusCode>`.
- Verified: array of integers, one per id. `-1` for an id that doesn't
  exist; `1` deleted; `2` with `ACKED` when the entry was not deleted
  because it is still referenced. Same shape in RESP2 and RESP3. Confirm the
  full meaning of each code against the Valkey docs and encode it as an enum.

**T4.5 `CONFIG INFO`** — agent — reviewers: @rlunar — new issue. Since
9.2.0. Read its docs from `COMMAND DOCS CONFIG|INFO` before writing tests.

**T4.6 `EVAL_RO`, `EVALSHA_RO`** — agent + design sign-off — reviewers:
both — #7880. Read-only scripts routed by `ReadFrom`; `EVALSHA_RO` reloads on
`NOSCRIPT` as `Script` does today. Since 7.0.0. Red: script on a replica
(`ReadFrom::PreferReplica`) succeeds with `_RO` and the write variant is
rejected.

**T4.7 Bloom (`BloomCommands`)** — agent + design sign-off — reviewers:
both — #4649 — depends on T1.8

- Syntax (valkey-bloom 1.0.1):
    - `BF.ADD key value`, `BF.EXISTS key value`, `BF.CARD key`
    - `BF.MADD key value [value …]`, `BF.MEXISTS key value [value …]`
    - `BF.RESERVE key error_rate capacity [EXPANSION n] [NONSCALING]`
    - `BF.INSERT key [CAPACITY n] [ERROR r] [EXPANSION n] [SEED s] [TIGHTENING t] [VALIDATESCALETO n] [NONSCALING] [NOCREATE] [ITEMS value [value …]]`
    - `BF.INFO key [CAPACITY | SIZE | FILTERS | EXPANSION | ERROR | TIGHTENING | MAXSCALEDCAPACITY]`
    - `BF.LOAD key dump`
- redis-rs names to match where the arguments fit: `bf_add`, `bf_exists`,
  `bf_card`, `bf_madd`, `bf_mexists`, `bf_reserve`, `bf_reserve_options`,
  `bf_insert`, `bf_insert_options`, `bf_info`, `bf_info_type`.
  `bf_scandump`/`bf_loadchunk` don't exist in valkey-bloom: record as
  differences; add a GLIDE `bf_load`.
- Verified replies (same in RESP2 and RESP3 unless noted):
    - `BF.ADD`/`BF.EXISTS`: integer 1/0 → `bool`.
    - `BF.MADD`/`BF.MEXISTS`/`BF.INSERT`: array of 1/0 → `Vec<bool>`.
    - `BF.CARD`: integer; `0` for a missing key.
    - `BF.RESERVE`: `OK`; existing key: error `ERR item exists`.
    - `BF.INSERT … NOCREATE` on a missing key: error `ERR not found`.
    - `BF.INFO key`: **an array of label/value pairs in both protocols**
      (labels `Capacity`, `Size`, `Number of filters`,
      `Number of items inserted`, `Error rate`, `Expansion rate`,
      `Tightening ratio`, `Max scaled capacity`). `Error rate` and
      `Tightening ratio` are bulk strings in RESP2 and doubles in RESP3.
      Decode into a typed struct.
    - `BF.INFO key ERROR`: bulk string (RESP2) or double (RESP3) → `f64`.
    - `BF.INFO` on a missing key: error `ERR not found`.
- Tests: `tests/modules_bloom.rs` (T1.8 harness), RESP2 and RESP3,
  standalone and cluster.

**T4.8 `JSON.MERGE`, `JSON.MSET`** — agent — reviewers: both — #5883 —
depends on T1.8

- Syntax: `JSON.MERGE key path json`; `JSON.MSET key path json [key path json …]`.
- Verified: both reply `OK`. `JSON.MERGE` with `null` for a member deletes
  it (`{"a":1,"b":{"c":2}}` merged with `{"b":{"d":3},"a":null}` gives
  `{"b":{"c":2,"d":3}}`).
- Cluster: `JSON.MSET` is multi-key. Note for @madolson: check glide-core
  multi-slot handling for `JSON.MSET` (#2580, #7780) before testing across
  slots.

**T4.9 Core search fixes** — agent — reviewers: @madolson — one PR each

- T4.9a #7593: `FT.SEARCH`/`FT.AGGREGATE` route to the index name's shard in
  cluster mode. Red: cluster test with an index whose data spans shards.
- T4.9b #7859: add `FT.HYBRID`, `FT.CURSOR` to the read-only list
  (`is_readonly_cmd` in `glide-core/redis-rs/redis/src/cluster_routing.rs`).
  Red: unit test on `is_readonly_cmd`.
- T4.9c #7884: `FT.SEARCH WITHCURSOR` reply conversion keeps the cursor id.
  Red: core conversion unit test with a captured reply.

**T4.10 Search (`SearchCommands`)** — agent + design sign-off — reviewers:
both — #7286 (retarget to valkey-search 1.3) — depends on T1.8, T4.9

- One PR each: T4.10a `FT.CREATE`, `FT.DROPINDEX`, `FT._LIST`, `FT.INFO`;
  T4.10b `FT.SEARCH`; T4.10c `FT.AGGREGATE`; T4.10d `FT.HYBRID`;
  T4.10e `FT.CURSOR READ`/`DEL`.
- Model arguments on valkey-search 1.3 (`src/commands/*.json` at tag
  `1.3.0`) and the Java implementation merged in #7379, not redis-rs's
  RediSearch-shaped `ft_create`.
- Cursor ids are `u64`.
- Do not add `FT.ALIAS*`, `FT.EXPLAIN*` or `FT.PROFILE` (not in
  valkey-search 1.3).

**T4.11 Path Hash core** — agent — reviewers: @madolson — Path Hash issue

- Add `PHGET`, `PHMGET`, `PHGETALL`, `PHPATHEXISTS`, `PHPATHCOUNT`,
  `PHPATHSCAN`, `PHSCAN`, `PHPREFIXES` to `is_readonly_cmd`.
- Normalize `PHGETALL` to a map (RESP2 is a flat array) and the values in
  `PHPREFIXES … WITHVALUES` to maps (RESP2 flat arrays) in
  `value_conversion.rs`.
- Red: unit tests on `is_readonly_cmd` and on conversion of captured RESP2
  replies.

**T4.12 Path Hash Rust** — agent + design sign-off — reviewers: both —
depends on T4.11. Behind an opt-in `unstable-pathhash` feature. Full spec
and verified behavior: `command-gap-analysis.md`, section "Path Hash".

- T4.12a writes and reads: `phset`, `phmset`, `phget`, `phmget` (regroup
  the flat reply per path), `phgetall`, `phdel`, `phpathdel`,
  `phpathexists`, `phpathcount`.
- T4.12b scans: `phscan` (numeric cursor) and `phpathscan` (opaque string
  cursor such as `1:2f75736572732f31`; `ScanIter` in
  `src/commands/scan.rs` stores `cursor: u64`, so add a byte-cursor
  iterator).
- T4.12c `PHPREFIXES` with one method per reply shape (plain, lengths,
  with values, fields).
- Edge cases to test: `FNX`/`FXX` all-or-nothing (`nil`, nothing written),
  byte-prefix matching (`/users` matches `/users2`), empty path, last field
  deleted removes the path, `WRONGTYPE`, `FIELDS 0` error.

**T4.13–T4.18 Management commands** — agent — reviewers: @rlunar
(+@madolson for cluster). Verify syntax with `COMMAND DOCS` on 9.2 before
writing tests.

| ID    | Commands                                                                                                                                                                           | Tracking            |
| ----- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------- |
| T4.13 | Read-only `CLUSTER`: `INFO`, `NODES`, `SHARDS`, `SLOTS`, `MYID`, `MYSHARDID`, `KEYSLOT`, `COUNTKEYSINSLOT`, `GETKEYSINSLOT`, `LINKS`, `REPLICAS`, `SLOT-STATS`                     | #7299               |
| T4.14 | `COMMANDLOG GET/LEN/RESET`, `SLOWLOG GET/LEN/RESET`, `HOTKEYS GET/RESET`, `MODULE LIST`, `CLIENT KILL/LIST/INFO`                                                                   | #6292, #7196        |
| T4.15 | `ACL` (17 redis-rs methods) and 9.2 roles: `SETROLE`, `GETROLE`, `DELROLE`, `ROLES`, `DIGEST`                                                                                      | #7060, new issue    |
| T4.16 | `CLUSTER` admin and atomic slot migration: `MIGRATESLOTS`, `GETSLOTMIGRATIONS`, `CANCELSLOTMIGRATIONS`, `FLUSHSLOT`                                                                | #7299, #6151        |
| T4.17 | `COMMAND COUNT/DOCS/INFO/LIST/GETKEYS/GETKEYSANDFLAGS`, `MEMORY USAGE/STATS/DOCTOR/MALLOC-STATS/PURGE`, `LATENCY HISTOGRAM`, `ROLE`, `SWAPDB`, `WAITAOF`, `SHUTDOWN`, `GEORADIUS*` | #6292, #7060        |
| T4.18 | Remaining #7060 methods (stream conveniences, `scan_options`, `zadd_options`, `invoke_script`, `load_script`), #7082 options, `incr_by_float` (#7262)                              | #7060, #7082, #7262 |

### Wave 5 — feature parity and infrastructure

| ID    | Task                                                                                                                              | Type                                            | Reviewers | Red test                                                                                                                         |
| ----- | --------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------- | --------- | -------------------------------------------------------------------------------------------------------------------------------- |
| T5.1  | Circuit breaker config (#6916)                                                                                                    | agent                                           | both      | Config lowering unit test; integration test that the breaker opens                                                               |
| T5.2  | Client-side caching (#6918)                                                                                                       | agent + design sign-off                         | both      | Cached `GET` served without a server round trip (assert via `INFO commandstats`); invalidation after a write from another client |
| T5.3  | Client info tag (#6920)                                                                                                           | agent                                           | @rlunar   | `CLIENT INFO` shows the tag                                                                                                      |
| T5.4  | Custom credentials provider (#6921)                                                                                               | agent + design sign-off                         | both      | Provider called on connect and on rotation                                                                                       |
| T5.5  | Custom logger (#6922)                                                                                                             | agent + design sign-off                         | @rlunar   | Logger receives core log records                                                                                                 |
| T5.6  | Value compression (#6878)                                                                                                         | agent                                           | both      | Value stored compressed (`STRLEN` smaller), read back equal                                                                      |
| T5.7  | OpenTelemetry command spans (#7699)                                                                                               | agent                                           | @rlunar   | File exporter receives a span per command                                                                                        |
| T5.8  | Unix sockets, standalone (#7645; draft PR #7451)                                                                                  | human coordinate, then agent                    | @rlunar   | Connect over a Unix socket                                                                                                       |
| T5.9  | `BigInt`/`BigUint`, optional `rust_decimal`/`bigdecimal` (#7664, #7663)                                                           | agent                                           | @rlunar   | Round-trip encode/decode tests per type                                                                                          |
| T5.10 | `test-util` feature: mock executor and argument assertions                                                                        | agent + design sign-off                         | @rlunar   | Downstream-style test using only public `test-util` API                                                                          |
| T5.11 | Cargo features in `glide-core`/`glide-telemetry` (`tls`, `tls-aws-lc`, `tls-ring`, `iam`, `telemetry`, `compression`), default-on | agent + design sign-off                         | both      | `cargo hack --feature-powerset` (T1.3) fails until gating compiles; every binding's CI green                                     |
| T5.12 | Parity guard beyond the command table (#7288)                                                                                     | agent                                           | @rlunar   | Guard detects a seeded missing `hotkeys_*` method                                                                                |
| T5.13 | Cargo workspace for published crates (#7261)                                                                                      | agent + design sign-off                         | both      | `cargo publish --workspace --dry-run`                                                                                            |
| T5.14 | Dedicated connections / pooling, then `MONITOR` (#6917, #5815, #6919)                                                             | human (after RFC #5815 is accepted), then agent | both      | —                                                                                                                                |
| T5.15 | Untested extension-trait commands; shared test matrix (#7236, #7289, #6996, #6997)                                                | agent                                           | @rlunar   | Coverage report lists the commands with no integration test                                                                      |

### Wave 6 — engine (glide-core; reviewers: @madolson + binding owners)

| ID    | Task                                                                                                                                                                                                                                                                                 | Type                                         | Red test                                                                               |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------- | -------------------------------------------------------------------------------------- |
| T6.1  | Stage 0a: delete fork code with no consumer, one module per PR (`sentinel.rs`; `aio/connection_manager.rs` plus the `connection-manager` feature in `glide-core/Cargo.toml`, `java/Cargo.toml`, `python/glide-async/Cargo.toml`; `cluster_pipeline.rs`; the `dummy-for-ort` package) | agent                                        | Each binding's CI must stay green; "red" is a grep test asserting no references remain |
| T6.2  | Stage 0b: rename the fork's lib from `redis` to `glide_core_engine`                                                                                                                                                                                                                  | agent + design sign-off                      | Build fails in every crate until imports are renamed                                   |
| T6.3  | Stage 1: `glide_core::protocol` facade; remove `glide-core-engine` from `ffi`, `java`, `node/rust-client`, `python/glide-async`, `rust`, `benchmarks/rust` manifests; CI check that only `glide-core` names the engine                                                               | agent + design sign-off                      | The CI check fails while any manifest still lists the engine                           |
| T6.4+ | Stages 2–4 (parser, connections, cluster move-then-refactor)                                                                                                                                                                                                                         | human-led design per stage, then agent tasks | Differential tests against the old implementation                                      |

### Wave 7 — GA (1.0)

| ID   | Task                                                                                                                                | Type  | Reviewers |
| ---- | ----------------------------------------------------------------------------------------------------------------------------------- | ----- | --------- |
| T7.1 | Add `valkey-glide` to `benchmarks/rust` (today it benchmarks `glide-core` directly) and compare with redis-rs; scenarios in the RFC | agent | both      |
| T7.2 | Docs refresh: README, `DESIGN.md`, `migration.md` (redis-rs 1.x and 2.x) (#7205, #7295)                                             | agent | @rlunar   |
| T7.3 | glide.valkey.io Rust quickstart and Rust tabs (docs repo has only `concepts/architecture/rust-core-design.mdx`)                     | agent | @rlunar   |
| T7.4 | `examples/rust/` (new): standalone, cluster, pipeline, pub/sub, IAM, search, bloom; built in CI                                     | agent | @rlunar   |
| T7.5 | Release candidate, bug bash, `cargo semver-checks` gate from 1.0; release through `rust-cd.yml`                                     | human | both      |

## 7. Order at a glance

```text
Wave 0 (human) ─► Wave 1 (parallel) ─► Wave 2 (T2.1 → T2.2 → T2.3/T2.4/T2.5)
                                              │
                         Wave 3 (parallel, after T2.2) ─► release 0.11
                                              │
     T1.8 ─► T4.7, T4.8, T4.10      T4.9 ─► T4.10      T4.11 ─► T4.12
     Wave 4 and Wave 5 in parallel after T2.2 and T2.5
     T6.1 → T6.2 → T6.3 (before 1.0) ─► T6.4+ (after 1.0)
                                              │
                                    Wave 7 ─► 1.0
```

Good first tasks for agents (small, isolated, clear red test): T1.1, T1.6,
T1.7, T3.8, T3.12, T4.5.
