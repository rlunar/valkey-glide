# Rust client command gap analysis

Goal: first-party, typed support in `valkey-glide` (Rust) for every
user-facing command in Valkey 9.2, valkey-search 1.3, valkey-json 1.1 and
valkey-bloom 1.0, so users never need `custom_command` or
`glide_send_command` for a documented command. Also: parity with the redis-rs
1.7 command table where the command exists in Valkey.

Companion to [rfc-redis-rs-free-rust-client.md](rfc-redis-rs-free-rust-client.md).

## Summary

| Source                 | Version checked | User-facing commands  | Typed in Rust today | Missing                                                 |
| ---------------------- | --------------- | --------------------- | ------------------- | ------------------------------------------------------- |
| Valkey core            | 9.2.0           | 425 incl. subcommands | 267                 | 158 (110 user-facing after the exclusions in section 4) |
| valkey-search          | 1.3.0           | 8 (+2 internal)       | 0                   | 8                                                       |
| valkey-json            | 1.1.0           | 24                    | 22                  | 2                                                       |
| valkey-bloom           | 1.0.1           | 9                     | 0                   | 9                                                       |
| redis-rs command table | 1.7.0           | 82 differences        | —                   | 75 methods (50 apply to Valkey, 25 are Redis-only)      |

Largest gaps, in order of user impact:

1. **Modules:** no typed search (`FT.*`) or bloom (`BF.*`) API at all;
   `JSON.MERGE` and `JSON.MSET` missing.
2. **Valkey 9.x data commands:** `DELEX`, `INCREX`, `MSETEX`, `XACKDEL`,
   `XDELEX`, the 12 path-hash commands (`PH*`), `EVAL_RO`/`EVALSHA_RO`.
3. **Management:** no typed `CLUSTER *`, `ACL *` (including 9.2 roles),
   `COMMANDLOG`, `HOTKEYS`, `SLOWLOG`, `MEMORY`, `MODULE`, `COMMAND *`,
   `CLIENT KILL`/`LIST`.

## Method

- **Valkey core:** started `valkey-server` 9.2.0 and read `COMMAND DOCS`
  (425 commands and subcommands, with `since` and group).
- **Modules:** command definition files (`src/commands/*.json`) at the
  release tags `valkey-search@1.3.0`, `valkey-json@1.1.0`,
  `valkey-bloom@1.0.1`. For search, cross-checked with the registered names in
  `src/commands/commands.h`.
- **Rust coverage:** a command counts as covered when its name (and, for
  subcommands, each token) appears as a string literal in `rust/src`,
  excluding `parity_tests/`, `mock_tests/` and `#[cfg(test)] mod tests`. False
  positives were checked by hand: `DECR` is sent as `DECRBY` (as in redis-rs)
  and `MULTI`/`EXEC` are sent by atomic pipelines, so both count as covered.
  Subcommands also had to have their tokens within a few lines of each other;
  that rule caught `MEMORY STATS`, `CLIENT INFO` and `CONFIG INFO` as false
  matches (they are not typed), while the `PUBSUB` subcommands are typed
  through a helper.
- **redis-rs parity:** `rust/src/parity_tests/differences.json`.
- **Upstream tracking:** `gh search issues` on valkey-io/valkey-glide for
  each command family.

Reproduce: start any Valkey 9.2 server, `valkey-cli --json COMMAND DOCS`,
and grep `rust/src` as above.

## 1. Modules (P0)

### valkey-search 1.3.0

Commands registered by the module: `FT.CREATE`, `FT.SEARCH`, `FT.AGGREGATE`,
`FT.HYBRID`, `FT.CURSOR` (`READ`/`DEL`), `FT.INFO`, `FT.DROPINDEX`,
`FT._LIST`, plus internal `FT._DEBUG` and `FT.INTERNAL_UPDATE` (not exposed).

| Command                | Rust | Python | Java | Node | Go  | Notes                                                                                    |
| ---------------------- | ---- | ------ | ---- | ---- | --- | ---------------------------------------------------------------------------------------- |
| `FT.CREATE`            | no   | yes    | yes  | yes  | yes | redis-rs `ft_create` targets RediSearch; model valkey-search's arguments, not redis-rs's |
| `FT.SEARCH`            | no   | yes    | yes  | yes  | yes |                                                                                          |
| `FT.AGGREGATE`         | no   | yes    | yes  | yes  | yes |                                                                                          |
| `FT.HYBRID`            | no   | yes    | no   | yes  | yes | New in 1.3; core request type added in #7456                                             |
| `FT.CURSOR READ`/`DEL` | no   | yes    | no   | yes  | yes | Core request types `FtCursorRead`/`FtCursorDel`                                          |
| `FT.INFO`              | no   | yes    | yes  | yes  | yes |                                                                                          |
| `FT.DROPINDEX`         | no   | yes    | yes  | yes  | yes |                                                                                          |
| `FT._LIST`             | no   | yes    | yes  | yes  | yes |                                                                                          |

Tracking: [#7286](https://github.com/valkey-io/valkey-glide/issues/7286)
(Rust FT support, written against redis-rs). Update it to target
valkey-search 1.3 instead.

Core bugs to fix before or with the Rust API, since a typed API exposes them:

- [#7593](https://github.com/valkey-io/valkey-glide/issues/7593): `FT.SEARCH`/`FT.AGGREGATE` always route to the index name's shard in cluster mode.
- [#7859](https://github.com/valkey-io/valkey-glide/issues/7859): `FT.HYBRID` and `FT.CURSOR` missing from the read-only list (breaks `ReadFrom`).
- [#7884](https://github.com/valkey-io/valkey-glide/issues/7884): `FT.SEARCH WITHCURSOR` reply conversion fails and the cursor id is lost.
- [#7858](https://github.com/valkey-io/valkey-glide/issues/7858) (Node only): cursor ids above 2^53. Rust should use `u64`.

Finding: GLIDE core defines request types for `FT.ALIASADD`/`ALIASDEL`/
`ALIASLIST`/`ALIASUPDATE`, `FT.EXPLAIN`, `FT.EXPLAINCLI` and `FT.PROFILE`, but
valkey-search 1.3.0 registers none of them. Do not add these to Rust unless a
supported target server implements them; confirm which server the other
bindings' versions target.

### valkey-bloom 1.0.1

`BF.ADD`, `BF.CARD`, `BF.EXISTS`, `BF.INFO`, `BF.INSERT`, `BF.LOAD`,
`BF.MADD`, `BF.MEXISTS`, `BF.RESERVE`.

- Missing in **every** GLIDE client:
  [#4649](https://github.com/valkey-io/valkey-glide/issues/4649).
- redis-rs 1.7 has 13 `bf_*` methods (`bf_add`, `bf_card`, `bf_exists`,
  `bf_info`, `bf_info_type`, `bf_insert`, `bf_insert_options`, `bf_loadchunk`,
  `bf_madd`, `bf_mexists`, `bf_reserve`, `bf_reserve_options`, `bf_scandump`).
  `BF.SCANDUMP`/`BF.LOADCHUNK` do not exist in valkey-bloom; it uses
  `BF.LOAD`. Record those two as deliberate differences.
- Rust can lead here: a `BloomCommands` extension trait, shaped like
  redis-rs's `bf_*` methods where the arguments match, and following
  valkey-bloom's options for `BF.RESERVE`/`BF.INSERT`.

### valkey-json 1.1.0

22 of 24 commands are typed (`JsonCommands` in `rust/src/commands/json.rs`).

| Missing      | Other clients  | Tracking                                                                                                          |
| ------------ | -------------- | ----------------------------------------------------------------------------------------------------------------- |
| `JSON.MERGE` | missing in all | none found                                                                                                        |
| `JSON.MSET`  | missing in all | [#5883](https://github.com/valkey-io/valkey-glide/issues/5883) (multi-key, possibly multi-slot; see #2580, #7780) |

## 2. Valkey 9.x data commands (P1)

| Command                                                              | Since   | Rust    | Tracking                                                                                                                                           | redis-rs method                      |
| -------------------------------------------------------------------- | ------- | ------- | -------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------ |
| `DELEX` (`IFEQ`/`IFNE`)                                              | 9.2     | no      | [#7238](https://github.com/valkey-io/valkey-glide/issues/7238), open PR [#7843](https://github.com/valkey-io/valkey-glide/pull/7843) includes Rust | `del_ex`                             |
| `INCREX`                                                             | 9.2     | no      | [#7200](https://github.com/valkey-io/valkey-glide/issues/7200)                                                                                     | `increx`                             |
| `MSETEX`                                                             | 9.1     | no      | [#6152](https://github.com/valkey-io/valkey-glide/issues/6152)                                                                                     | `mset_ex`                            |
| `XACKDEL`, `XDELEX` (`KEEPREF`/`DELREF`/`ACKED`)                     | 9.2     | no      | [#7197](https://github.com/valkey-io/valkey-glide/issues/7197)                                                                                     | `xack_del`, `xdel_ex`                |
| Path hash: 12 `PH*` commands (see [Path Hash](#path-hash-valkey-92)) | 9.2     | no      | **none** (new data type; no issue in any client)                                                                                                   | none                                 |
| `EVAL_RO`, `EVALSHA_RO`                                              | 7.0     | no      | [#7880](https://github.com/valkey-io/valkey-glide/issues/7880) (with `ReadFrom` routing)                                                           | none                                 |
| `CONFIG INFO`                                                        | 9.2     | no      | none found                                                                                                                                         | none                                 |
| `SET ... IFNE`                                                       | 9.2     | yes     | done (#7453)                                                                                                                                       |                                      |
| `HGETDEL`, `DELIFEQ`                                                 | 9.0/9.1 | yes     | done (#7800)                                                                                                                                       |                                      |
| `HSETEX` `NX`/`XX`                                                   | 9.0     | partial | [#7873](https://github.com/valkey-io/valkey-glide/issues/7873), PR [#7874](https://github.com/valkey-io/valkey-glide/pull/7874)                    |                                      |
| `GEORADIUS`, `GEORADIUSBYMEMBER` (+`_RO`), `SUBSTR`, `HMSET`         | old     | no      | #7060 for the geo ones                                                                                                                             | `geo_radius`, `geo_radius_by_member` |

`GEORADIUS*`, `SUBSTR` and `HMSET` are **not** flagged deprecated in Valkey
9.2's `COMMAND DOCS` (only `SLAVEOF` and `CLUSTER SLAVES` are), so they count
as gaps. `SUBSTR` and `HMSET` duplicate `GETRANGE` and `HSET`; adding them is
low value and optional.

### Path Hash (Valkey 9.2)

A new key type (`TYPE` and `OBJECT ENCODING` return `pathhash`) backed by
Valkey's radix tree. Each key maps binary-safe **paths** to a
`field → value` map. Its distinguishing operation is `PHPREFIXES`, which
returns every stored ancestor of a query path, longest first, in one call.
Introduced by [valkey#4506](https://github.com/valkey-io/valkey/pull/4506)
(merged 2026-09-16), whose first target use case is a KV-cache placement
index for AI inference routing. The 9.2 release post also lists hierarchical
routing, longest-prefix matching and autocomplete.

No GLIDE client supports it, and there is no GLIDE issue for it.

#### Commands

| Command        | Syntax                                                                                                                     | Flags          |
| -------------- | -------------------------------------------------------------------------------------------------------------------------- | -------------- |
| `PHSET`        | `PHSET key path [FNX \| FXX] FIELDS numfields field value [field value ...]`                                               | write, fast    |
| `PHMSET`       | `PHMSET key path FIELDS numfields field value [...] [path FIELDS ...]`                                                     | write          |
| `PHGET`        | `PHGET key path field [field ...]`                                                                                         | readonly, fast |
| `PHMGET`       | `PHMGET key path FIELDS numfields field [...] [path FIELDS ...]`                                                           | readonly       |
| `PHGETALL`     | `PHGETALL key path`                                                                                                        | readonly       |
| `PHDEL`        | `PHDEL key path [field ...]` (no fields deletes the whole path)                                                            | write          |
| `PHPATHDEL`    | `PHPATHDEL key path [WITHSUBPATHS]`                                                                                        | write          |
| `PHPATHEXISTS` | `PHPATHEXISTS key path`                                                                                                    | readonly       |
| `PHPATHCOUNT`  | `PHPATHCOUNT key [PREFIX prefix]`                                                                                          | readonly       |
| `PHPATHSCAN`   | `PHPATHSCAN key cursor [PREFIX prefix] [COUNT count]`                                                                      | readonly       |
| `PHSCAN`       | `PHSCAN key path cursor [MATCH pattern] [COUNT count]`                                                                     | readonly       |
| `PHPREFIXES`   | `PHPREFIXES key query [LENGTHS] [WITHVALUES \| FIELDS numfields field ...] [COUNT count] [MAXPATHLEN bytes] [EXCLUDESELF]` | readonly       |

All commands take one key at position 1, so cluster routing is by that key.
ACL category: `@pathhash`.

#### Behavior verified against Valkey 9.2.0

Run with raw RESP2 and RESP3 connections:

| Behavior                        | Result                                                                                                                                  |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| `PHSET` reply                   | `OK`, or `nil` when `FNX`/`FXX` fails                                                                                                   |
| `FNX`/`FXX` with several fields | All-or-nothing: one failing field means `nil` and nothing is written                                                                    |
| `PHGET`, `PHMGET`               | Array with `nil` for missing fields. `PHMGET` returns one **flat** array across all path groups                                         |
| `PHGETALL`                      | RESP2: flat array `[f, v, …]`. RESP3: map. Missing path: empty                                                                          |
| `PHPREFIXES` (plain)            | Array of paths, longest first                                                                                                           |
| `PHPREFIXES LENGTHS`            | Array of integers (prefix lengths in bytes)                                                                                             |
| `PHPREFIXES WITHVALUES`         | Array of `[path, values]`. `values` is a flat array in RESP2 and a map in RESP3                                                         |
| `PHPREFIXES FIELDS …`           | Array of `[path, [value \| nil, …]]`                                                                                                    |
| `PHPATHSCAN` cursor             | **Opaque string** (for example `1:2f75736572732f31`), not an integer; `0` ends the scan                                                 |
| `PHSCAN` cursor                 | Numeric string, like `HSCAN`; no `NOVALUES` option                                                                                      |
| Prefix semantics                | Raw byte prefixes, not path segments: `PREFIX /users` also matches `/users2`, and `PHPREFIXES /users2/x` returns `/users2` and `/users` |
| Paths and fields                | Binary-safe. Empty path and empty field name are accepted. Paths need not start with `/`                                                |
| Empty paths and keys            | Deleting the last field of a path removes the path; removing the last path removes the key                                              |
| Whole-key commands              | `EXPIRE`/`TTL`, `COPY`, `RENAME`, `DUMP`, `MEMORY USAGE`, `SCAN … TYPE pathhash` work. No per-path or per-field expiry                  |
| Errors                          | `WRONGTYPE` on other key types; `PHSET … FIELDS 0` is a wrong-number-of-arguments error                                                 |

#### Upstream API status

[valkey#4706](https://github.com/valkey-io/valkey/issues/4706) tracks open
decisions after the merge. Ones that could change a client API:

- Key-level `NX`/`XX` and path-level `PNX`/`PXX` on writes (not decided).
- Making `PHDEL` field-only (whole-path deletion would then be `PHPATHDEL`
  only) and a multi-path delete.
- A field-level existence command (`PHEXISTS key path field`).
- Final naming of `PHMSET`/`PHMGET`.
- A relative-path batch insert (`PHCHAINSET` or a `PHMSET` extension).

Additive changes (new options, new commands) fit a minor GLIDE release.
Making `PHDEL` field-only or renaming `PHMSET`/`PHMGET` would be breaking.

#### Work needed

**glide-core (all bindings):**

1. Add the read commands (`PHGET`, `PHMGET`, `PHGETALL`, `PHPATHEXISTS`,
   `PHPATHCOUNT`, `PHPATHSCAN`, `PHSCAN`, `PHPREFIXES`) to
   `is_readonly_cmd` (in the fork, `glide-core/redis-rs/redis/src/cluster_routing.rs`), so `ReadFrom` can route them to replicas. Today the
   list has no `PH*` entry.
2. Normalize RESP2/RESP3 in `glide-core/src/client/value_conversion.rs`: `PHGETALL` to a map (as
   `HGETALL` already is), and the values in `PHPREFIXES WITHVALUES` to maps.
   Without this every binding returns different shapes per protocol.

**Rust (`valkey-glide`):** a `PathHashCommands` extension trait. Sketch:

```rust,ignore
pub trait PathHashCommands {
    async fn phset(&self, key, path, fields: &[(F, V)], cond: Option<FieldCondition>) -> ValkeyResult<bool>;
    async fn phmset(&self, key, entries: &[(P, &[(F, V)])]) -> ValkeyResult<()>;
    async fn phget(&self, key, path, fields: &[F]) -> ValkeyResult<Vec<Option<Bytes>>>;
    async fn phmget(&self, key, groups: &[(P, &[F])]) -> ValkeyResult<Vec<Vec<Option<Bytes>>>>;
    async fn phgetall(&self, key, path) -> ValkeyResult<HashMap<Bytes, Bytes>>;
    async fn phdel(&self, key, path, fields: &[F]) -> ValkeyResult<u64>;
    async fn phpathdel(&self, key, path, with_subpaths: bool) -> ValkeyResult<u64>;
    async fn phpathexists(&self, key, path) -> ValkeyResult<bool>;
    async fn phpathcount(&self, key, prefix: Option<P>) -> ValkeyResult<u64>;
    fn phpathscan(&self, key, options: PathScanOptions) -> ScanIter<Bytes>;
    fn phscan(&self, key, path, options: ScanOptions) -> ScanIter<(Bytes, Bytes)>;
    async fn phprefixes(&self, key, query, options: PrefixesOptions) -> ValkeyResult<Vec<Bytes>>;
    async fn phprefixes_lengths(&self, key, query, options: PrefixesOptions) -> ValkeyResult<Vec<u64>>;
    async fn phprefixes_with_values(&self, key, query, options: PrefixesOptions)
        -> ValkeyResult<Vec<(Bytes, HashMap<Bytes, Bytes>)>>;
    async fn phprefixes_fields(&self, key, query, fields: &[F], options: PrefixesOptions)
        -> ValkeyResult<Vec<(Bytes, Vec<Option<Bytes>>)>>;
}
```

Design notes:

- Paths, fields and values return as `Bytes`: the type is binary-safe and
  the primary use case stores block hashes, not text. Callers decode to
  `String` where they know the data is text.
- `phmget` regroups the server's flat reply using the request's field
  counts, so callers get one `Vec` per path.
- `PHPREFIXES` has one method per reply shape, so each return type is exact
  instead of a `ValkeyValue` the caller must inspect. `PrefixesOptions` holds
  `COUNT`, `MAXPATHLEN` and `EXCLUDESELF`.
- `phpathscan` needs a `Bytes` cursor. `ScanIter` (`src/commands/scan.rs`)
  stores `cursor: u64` and decodes each page as `(u64, Vec<RV>)`, so it would
  fail on `PHPATHSCAN`'s `1:2f75…` cursors. Generalize the cursor type or add
  a separate iterator.
- `FieldCondition` (`OnlyIfNoneExist` → `FNX`, `OnlyIfAllExist` → `FXX`) and
  all option structs are `#[non_exhaustive]`, so `PNX`/`PXX` or key-level
  conditions can be added later without a break.
- Docs state the byte-prefix semantics explicitly, since `/users` matching
  `/users2` surprises users who think in path segments.
- Ship behind an opt-in `unstable-pathhash` Cargo feature, documented as
  tracking an evolving server API, until valkey#4706's API items close. Then
  make it default.
- Tests: `matrix_test!` over standalone and cluster, RESP2 and RESP3, skipped
  below Valkey 9.2. Cover `FNX`/`FXX` all-or-nothing, byte-prefix matching,
  opaque cursor round trip, empty path, and `WRONGTYPE`.

**Other bindings:** the same commands in Python, Java, Node and Go, using
the glide-core normalization. Rust can go first and set the shape, as with
bloom.

**Estimate:** glide-core about 1 day; Rust trait, options, tests and docs
3–4 days; each other binding 2–3 days.

## 3. Management and cluster commands (P2)

| Family                         | Missing in Rust                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Tracking                                                                                                                                                                                       | Notes                                                                                                                                                                                        |
| ------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `CLUSTER` (all 37 subcommands) | `INFO`, `NODES`, `SHARDS`, `SLOTS`, `MYID`, `MYSHARDID`, `KEYSLOT`, `COUNTKEYSINSLOT`, `GETKEYSINSLOT`, `LINKS`, `REPLICAS`, `SLOT-STATS` (8.0), plus admin (`ADDSLOTS[RANGE]`, `DELSLOTS[RANGE]`, `FAILOVER`, `FORGET`, `MEET`, `REPLICATE`, `RESET`, `SETSLOT`, `SAVECONFIG`, `SET-CONFIG-EPOCH`, `BUMPEPOCH`, `COUNT-FAILURE-REPORTS`, `FLUSHSLOTS`) and atomic slot migration (9.0: `MIGRATESLOTS`, `GETSLOTMIGRATIONS`, `CANCELSLOTMIGRATIONS`, `FLUSHSLOT`) | [#7299](https://github.com/valkey-io/valkey-glide/issues/7299), [#6151](https://github.com/valkey-io/valkey-glide/issues/6151), [#6292](https://github.com/valkey-io/valkey-glide/issues/6292) | Read-only subcommands first; admin ones are rarely used from an application. Java is the only binding with most of them.                                                                     |
| `CLUSTERSCAN` (server-side)    | yes                                                                                                                                                                                                                                                                                                                                                                                                                                                               | [#6153](https://github.com/valkey-io/valkey-glide/issues/6153)                                                                                                                                 | Could back `cluster_scan` on 9.1+ servers.                                                                                                                                                   |
| `ACL`                          | all 19, including 9.2 roles (`SETROLE`, `GETROLE`, `DELROLE`, `ROLES`, `DIGEST`)                                                                                                                                                                                                                                                                                                                                                                                  | #7060 for the pre-9.2 ones (17 redis-rs methods); **no issue for 9.2 roles**                                                                                                                   |                                                                                                                                                                                              |
| `COMMANDLOG` (8.1)             | `GET`, `LEN`, `RESET`                                                                                                                                                                                                                                                                                                                                                                                                                                             | #6292, #7299                                                                                                                                                                                   | Valkey's replacement for slowlog and large-request logs.                                                                                                                                     |
| `HOTKEYS` (9.2)                | `GET`, `RESET`                                                                                                                                                                                                                                                                                                                                                                                                                                                    | [#7196](https://github.com/valkey-io/valkey-glide/issues/7196)                                                                                                                                 | redis-rs has `hotkeys_*` outside the command table, which the parity guard does not see (#7288). Valkey 9.2 now has the command, so these become implementable rather than a documented gap. |
| `SLOWLOG`                      | `GET`, `LEN`, `RESET`                                                                                                                                                                                                                                                                                                                                                                                                                                             | #6292                                                                                                                                                                                          |                                                                                                                                                                                              |
| `CLIENT`                       | `KILL`, `LIST`, `INFO`, `UNBLOCK`, `TRACKINGINFO`, `REPLY`                                                                                                                                                                                                                                                                                                                                                                                                        | #6292                                                                                                                                                                                          | `TRACKING`/`CACHING`/`GETREDIR` are driven by client-side caching (#6918), not typed APIs.                                                                                                   |
| `COMMAND`                      | `COUNT`, `DOCS`, `INFO`, `LIST`, `GETKEYS`, `GETKEYSANDFLAGS`                                                                                                                                                                                                                                                                                                                                                                                                     | #6292                                                                                                                                                                                          |                                                                                                                                                                                              |
| `MEMORY`                       | `USAGE`, `STATS`, `DOCTOR`, `MALLOC-STATS`, `PURGE`                                                                                                                                                                                                                                                                                                                                                                                                               | #6292                                                                                                                                                                                          |                                                                                                                                                                                              |
| `MODULE`                       | `LIST`, `LOAD`, `LOADEX`, `UNLOAD`                                                                                                                                                                                                                                                                                                                                                                                                                                | #6292                                                                                                                                                                                          | `MODULE LIST` is useful for feature detection of the three modules.                                                                                                                          |
| Other                          | `LATENCY HISTOGRAM`, `ROLE`, `SWAPDB`, `WAITAOF`, `SHUTDOWN`                                                                                                                                                                                                                                                                                                                                                                                                      | #6292                                                                                                                                                                                          |                                                                                                                                                                                              |
| `MONITOR`                      | yes                                                                                                                                                                                                                                                                                                                                                                                                                                                               | [#6919](https://github.com/valkey-io/valkey-glide/issues/6919)                                                                                                                                 | Needs a dedicated connection (see RFC, pooling).                                                                                                                                             |

## 4. Excluded on purpose

These do not need typed APIs:

- **Handled by `glide-core`:** `HELLO`, `AUTH` flows, `ASKING`, `READONLY`,
  `READWRITE`, `QUIT`, `CLIENT CAPA`, `CLIENT SETINFO`, `CLIENT TRACKING`,
  `CLIENT CACHING`, `CLIENT GETREDIR`, `MULTI`/`EXEC`/`DISCARD` (atomic
  pipelines).
- **Replication and cluster internals:** `PSYNC`, `REPLCONF`,
  `RESTORE-ASKING`, `CLUSTER SYNCSLOTS`, `CLIENT IMPORT-SOURCE`.
- **Debug and help:** `DEBUG`, `PFDEBUG`, `PFSELFTEST`, every `* HELP`
  subcommand, `SCRIPT DEBUG`.
- **Deprecated in Valkey 9.2:** `SLAVEOF`, `CLUSTER SLAVES` (use
  `REPLICAOF`, `CLUSTER REPLICAS`).

## 5. redis-rs 1.7 parity

75 methods in the redis-rs command table have no GLIDE implementation.

**Apply to Valkey (50), mostly under #7060:**

| Group          | Methods                                                                                                                                                                                                                                                     |
| -------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ACL (17)       | `acl_cat`, `acl_cat_categoryname`, `acl_deluser`, `acl_dryrun`, `acl_genpass`, `acl_genpass_bits`, `acl_getuser`, `acl_help`, `acl_list`, `acl_load`, `acl_log`, `acl_log_reset`, `acl_save`, `acl_setuser`, `acl_setuser_rules`, `acl_users`, `acl_whoami` |
| Bloom (13)     | `bf_*` (see section 1; `bf_scandump`/`bf_loadchunk` become deliberate differences)                                                                                                                                                                          |
| Valkey 9.x (5) | `del_ex` (#7238), `increx`, `mset_ex`, `xack_del`, `xdel_ex`                                                                                                                                                                                                |
| Streams (7)    | `xadd_map`, `xadd_maxlen`, `xadd_maxlen_map`, `xrange_all`, `xrange_count`, `xrevrange_all`, `xrevrange_count`                                                                                                                                              |
| Other (8)      | `geo_radius`, `geo_radius_by_member`, `scan_options`, `zadd_options`, `zadd_multiple_options` (#7082), `invoke_script`, `load_script`, `ft_create` (#7286; model valkey-search instead)                                                                     |

**Redis-only, keep as permanent differences (25):**

- Vector sets (17): `vadd`, `vadd_options`, `vcard`, `vdelattr`, `vdim`,
  `vemb`, `vemb_options`, `vgetattr`, `vinfo`, `vlinks`,
  `vlinks_with_scores`, `vrandmember`, `vrandmember_multiple`, `vrem`,
  `vsetattr`, `vsim`, `vsim_options`. Valkey's equivalent is valkey-search
  vector indexes.
- `BITOP` `DIFF`/`DIFF1`/`ANDOR`/`ONE` (`bit_diff`, `bit_diff1`,
  `bit_and_or`, `bit_one`): Valkey 9.2 `BITOP` supports only `AND`, `OR`,
  `XOR`, `NOT`.
- `digest`, `xcfgset`, `xnack`, `xinfo_stream_with_idempotency`: no such
  commands or options in Valkey 9.2.

Change the `differences.json` reason for these from "TODO #7060: not
implemented yet" to a Redis-only reason, so #7060's remaining scope is
accurate. The RFC freezes the parity target at redis-rs 2.0; re-run this
section against the 2.0 snapshot when #7641 lands.

## 6. Upstream tracking gaps

1. **Rust is missing from the cross-language trackers.** #6292 (Valkey
   Commands Tracker) and #7299 (Missing Commands for 2.8) list Python, Java,
   Node, Go, C# and PHP, not Rust. Add a Rust column.
2. **No issue exists for:** the 9.2 path-hash commands (`PH*`), the 9.2 ACL
   role commands, `CONFIG INFO` (9.2) and `JSON.MERGE`.
3. **#7286** targets redis-rs's RediSearch-shaped `ft_create`; retarget it to
   valkey-search 1.3.
4. **Core FT request types** for alias/explain/profile have no valkey-search
   1.3 backing; decide whether to keep them.

## 7. Suggested order and estimates

Estimates are for one engineer familiar with the crate, including typed
options, unit encoding tests, `matrix_test!` integration tests and docs, as
`rust/docs/dev/options.md` requires.

| Order | Work                                                                                                                                                    | Estimate  |
| ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------- | --------- |
| 1     | Land PR #7843 (`DELEX`), then `INCREX`, `MSETEX`, `XACKDEL`, `XDELEX`                                                                                   | 2–3 days  |
| 2     | `BloomCommands` (9 commands, CI needs a valkey-bloom server)                                                                                            | 3–4 days  |
| 3     | `JSON.MERGE`, `JSON.MSET` (`MSET` needs multi-slot handling in core)                                                                                    | 1–2 days  |
| 4     | Search: fix #7593, #7859, #7884 in core, then `SearchCommands` (8 commands; `FT.CREATE`, `FT.SEARCH`, `FT.AGGREGATE` and `FT.HYBRID` options are large) | 2–3 weeks |
| 5     | Path Hash: 2 glide-core fixes, then the Rust trait behind `unstable-pathhash` (see the Path Hash section)                                               | 1 week    |
| 6     | Read-only `CLUSTER` subcommands, `COMMANDLOG`, `SLOWLOG`, `HOTKEYS`, `MODULE LIST`, `CLIENT KILL`/`LIST`, `EVAL_RO`/`EVALSHA_RO`                        | 1 week    |
| 7     | `ACL` including roles, remaining `CLUSTER` admin and slot migration, `COMMAND`, `MEMORY`, other server commands                                         | 1–2 weeks |

Module CI: the Rust integration tests start their own servers
(`tests/common/server.rs` spawns `valkey-server` for standalone;
`tests/common/cluster.rs` uses `utils/cluster_manager.py` for cluster) and
cannot point at external endpoints. Module tests need endpoint support in the
harness and a workflow like `node-modules.yml`, which starts
`valkey/valkey-bundle` through `.github/actions/start-valkey-docker`. Use
`valkey-bundle:9.2.0` (valkey-json 1.1.0, valkey-bloom 1.0.1,
valkey-search 1.3.0); the action defaults to `9.1.3`, and the current 9.1 bundle (9.1.5) ships valkey-search 1.2.1.
