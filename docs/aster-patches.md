# Aster patch ledger: iroh-util

This repository is Aster's fork of [n0-computer/iroh-util](https://github.com/n0-computer/iroh-util). Aster carries the patches listed below on top of upstream. Upstream maintainers are welcome to incorporate any of them, in whole or in part, at their discretion.

## Branch and tag model

- `main` is upstream `main` plus one commit: the Aster README preamble and this ledger. Nothing is built from `main`. It is refreshed with `git merge upstream/main` and is never reset or force-pushed. Ledger edits are commits on `main`, never on `upgrade/*`.
- `upgrade/<repo>-v<upstream version>` (for example `upgrade/iroh-v1.4.0`, `upgrade/iroh-blobs-v0.103.1`) is the patched integration line: the upstream release plus the patches in this ledger. A new upstream base always gets a new branch, cut from the upstream release tag; Aster-only fixes and re-pins append to the current branch. Per-minor names (`upgrade/iroh-blobs-v0.103`, `upgrade/salvo-v1.0`, …) are historical. Consumers pin commits on it. Published `upgrade/*` branches and tags are never rewritten; a failed attempt is superseded by new commits and a new version.
- Topic branches carry one change each, are named for the change (for example `feat/read-into`, `security/plumtree-bounds`), are based on the upstream release tag (or, for a patch that builds on another, on that prerequisite patch's commit, which its row names), and are cherry-picked with `git cherry-pick -x` onto the upgrade branch, so every upgrade-branch commit names its source. A topic branch a row names is not rewritten; re-cutting it onto a new upstream base takes a versioned name, `<topic>-v<upstream version>`.
- Version (option E, from 2026-10-25): a release with Aster patches publishes as `major.minor.P` with `P = 1000 * (upstream_patch + 1) + n`, where `n` (1 to 999) counts Aster patch releases on that upstream version (upstream base 0.6.0 plus the first is `0.6.1001`), so it never collides with a crates.io release. Canonical rule: `_internal/docs/aster-rpc/261003_fork-pin-upgrade-runbook.md`; `publish-native-stack.py check` enforces it. Versions already published (0.6.3 and earlier) are grandfathered. A re-pin without a code change keeps the version and takes the tag suffix `-p<n>`.
- Tags: `aster-<repo>-v<published version>` (re-pins `aster-<repo>-v<published version>-p<n>`), created only after the Aster consumer trial passes against those exact commits, because tags cannot be moved.
- The fork manifest is `iroh-fork-manifest.toml` in the Aster repository (the only copy).
- Aster does not submit patches upstream; upstream maintainers are welcome to take any patch here.

Historical and current tag schemes in this repository:

- `aster-iroh-util-v<version>`: `aster-iroh-util-v0.6.1`, `-v0.6.2`, `-v0.6.3`, `-v0.7.1001`.
- `aster-iroh-util-v<version>-p<n>`, a later commit at the same crate version: `aster-iroh-util-v0.6.3-p1`, `-v0.6.3-p2`, `-v0.6.3-p3`, `-v0.6.3-p4` (patch-block re-pins with no source change).

## Current release

| Field | Value |
|---|---|
| Upstream base | `v0.7.0` (`1003b74`), the first upstream release with the single-actor `ConnectionPool` (#5, with #12 squashed in) and slot reservation (#18) |
| Integration branch | `upgrade/iroh-util-v0.7.0` |
| Tag | `aster-iroh-util-v0.7.1001` |
| Pinned commit | `bfef8edbbe599bc4ea892860ea5218cf4a006844` |
| Crate version | 0.7.1001 |
| Source repository | `https://forge.emrul.dev/Aster/iroh-util.git` |

The fork's own `[patch.crates-io]` pins iroh, iroh-base and iroh-relay to `9ba1447d` (Aster iroh 1.4.2002) and noq, noq-udp and noq-proto to `379f73677c` (Aster noq 1.4.1001), the stack Aster links it against (`168bc6f`, unchanged revisions from 0.6.3-p7). Upstream 0.7 marks `Options` and the error enums `#[non_exhaustive]` (#16, `85ae65e`): callers build `Options` from `Default` and set fields, as Aster's core does since this release.

Bootstrap pin line for source-tree builds (consumers install a versioned Aster SDK release instead and must not copy this):

```toml
iroh-util = { git = "https://forge.emrul.dev/Aster/iroh-util.git", rev = "bfef8edbbe599bc4ea892860ea5218cf4a006844" }
```

Where this fork lives: iroh-util was a checksum-verified registry bridge until 2026-08-25, when it was promoted to a fork for the connection-pool deadlock (row 24). It first lived at `github.com/aster-rpc/iroh-util`, until the Forgejo org repository existed. It then moved to Forgejo because it was the last fork not there, and the only one whose fetch could be refused by someone else: GitHub rate-limits anonymous git per source address, and when the Forge host crossed that limit the release stopped here, reporting "could not read Username for 'https://github.com'", which reads like a private repository rather than a throttled public one. The Forgejo repository is the same history (same commit ids, same tags), and the Aster publisher (`publish-native-stack.py`) checks out the pinned commit and fails unless `git rev-parse HEAD` matches, so provenance rests on the hash, not the hostname. Forgejo push-mirrors it to `github.com/getaster/iroh-util`, the public copy.

## Patches

Row numbers are stable patch ids shared across Aster's iroh-family forks (iroh, iroh-blobs, iroh-docs, iroh-gossip, iroh-util, noq), so a commit message that says "row 33" resolves here.

| Id | Topic branch | What | Why / what in Aster depends on it | Commits (topic / upgrade) | Status | Conflict or porting notes |
|---|---|---|---|---|---|---|
| 24 | `test/connection-pool-witnesses-v0.7.0` (tip `db2c4dc`, on `v0.7.0`; the 0.6 topic was `test/connection-pool-witnesses` `c6ca9970a3` on `cpfix` `aef1db3`) | Three Aster witness tests for the single-actor `ConnectionPool`: a 300-connect burst resolves under saturation; dropping the pool terminates the actor and closes held connections; slow-peer backlogs do not starve an unrelated peer. File: `src/connection_pool.rs` (tests only). Until 0.7.1001 the row also carried upstream's then-unreleased rewrite (PR #5, base `cpfix` `aef1db3`), replacing the 0.6.0 design whose two blocking-send cycles deadlocked the pool at more than 100 concurrent `get_or_connect` calls (portal-sync's freeze-apply-window wedge). | Every portal-sync blob fetch and docs sync dial goes through this pool; the witnesses pin the deadlock and starvation behaviour Aster depends on. | `db2c4dc` / `a180db4` (0.6: `aef1db3` + `c6ca9970a3` / `396759d` + `afed9e6`, released as 0.6.2) | carried (tests only) | The rewrite is upstream in v0.7.0 (`a1e54dc`, #5 with #12 squashed in; `603d890`, #18 slot reservation; `0f1b5d5`, #17), so only the witness tests are carried. The 0.6 topic branch was reconstructed after release, so its `(cherry picked from …)` trailers point at the upgrade-branch commits (see History). |
| 33 | `fix/pool-retire-connection-v0.7.0` (tip `be02c3d`, on `v0.7.0`; the 0.6 topic was `fix/pool-retire-connection` `5b67790` on `cpfix` `aef1db3`) | Adds `ConnectionPool::retire(&ConnectionRef)`: closes the connection with reason `retired` and forgets it, only while the pool still holds that same connection (matched by upstream's `ConnId`), so the next `get_or_connect` dials anew and a late or repeated retire never closes the replacement or a dial still running. It also takes the connection out of the unused set. `close(id)` is unchanged. File: `src/connection_pool.rs`. | Root cause 3 of the Aster restart/reconnect investigation (Aster repository, `docs/_internal/investigations/2026-10-15-restart-reconnect.md`): the iroh-blobs downloader (iroh-blobs fork row 34, which requires 0.7.1001 since iroh-blobs 0.104.2001) must drop its pooled connection to a provider that restarted on a new port without killing a fresh replacement. | `be02c3d` / `4ef2dca` (0.6: `5b67790` / `10c23d8`, released as 0.6.3 at `6c42d23`) | re-cut | **Aster's own; no upstream equivalent in v0.7.0** (upstream draft #13 adopts incoming connections and marks the older one `superseded`, a different trigger). Re-expressed on upstream's `ConnId`/`UnusedSet` (#5/#12): "remove the peer's entry if its `ConnId` matches, take it out of the unused set, close with `retired`"; Aster's own generation plumbing is dropped. Same signature and error type as 0.6.3. Tests: `retire_dials_anew_and_spares_the_replacement` (fails if retire ignores the id) and, new, `retire_removes_an_unused_connection_and_spares_a_pending_dial`. |

### Upstreamed or dropped

| Patch | Commit / tag | Status | When and why |
|---|---|---|---|
| Row 24's adoption of upstream's single-actor rewrite (PR #5 at `cpfix` `aef1db3`) | `396759d` (0.6.2) | upstreamed | Released upstream in v0.7.0 (`a1e54dc`, with #12; #18 `603d890`). Dropped from the stack in 0.7.1001 (2026-11-09); the row keeps the witness tests. |
| Independent Aster fix for the 0.6.0 connection-pool deadlock (0.6.1) | `fc724bd`, tag `aster-iroh-util-v0.6.1` | dropped | Published to the registry but never consumed; superseded on 2026-08-25 by 0.6.2, which adopts upstream's PR #5 rewrite instead (row 24). The tag is preserved for the record. Do not consume or republish it. |

## Porting to a new upstream release

Order across the iroh family: noq, then iroh, then the satellites in the Aster publisher's order: this repository (its patch block names noq and iroh by revision), iroh-gossip, iroh-blobs (row 34 needs `ConnectionPool::retire` from row 33), iroh-docs.

Before porting, check upstream:

1. Since v0.7.0 the base is an upstream release tag; row 24 is tests only.
2. Check whether upstream has gained a retire-by-identity API (row 33); if so, map `ConnectionPool::retire` onto it.

```bash
git fetch --all --tags --prune
git push origin <upstream-tag>          # mirror the upstream release tag (e.g. iroh-util-v0.7.0) to the forge
git switch -c upgrade/iroh-util-v<upstream version> <upstream-tag>
# re-point [patch.crates-io] at the new iroh / noq fork revs (iroh, iroh-base, iroh-relay; noq, noq-udp, noq-proto), commit
git cherry-pick -x db2c4dc        # row 24 witness tests
git cherry-pick -x be02c3d        # row 33
# version bump, only if the version rule requires it (Aster's registry has already
# served <upstream version> or higher in the series, as it has 0.6.1..0.6.3):
# set the crate to the next option E version (`1000 * (upstream_patch + 1) + n`), then
git commit -am "chore: release <published version>"
git push origin upgrade/iroh-util-v<upstream version>
# do not tag yet: tag after the consumer trial (below)
```

Prefer `cherry-pick -x`. Published `upgrade/*` branches and tags are never rewritten; a failed attempt is superseded by new commits and a new version.

An Aster-only fix on the current base is a topic branch, cherry-picked `-x` onto the current upgrade branch, followed by a version-bump commit to the next option E version (`1000 * (upstream_patch + 1) + n`) (as 0.6.3 was cut for row 33). A re-pin without a code change is a commit on the current upgrade branch with no version bump, tagged `aster-iroh-util-v<current version>-p<n>` after the consumer trial (as `aster-iroh-util-v0.6.3-p1` was).

Validate in this repository:

```bash
cargo fmt --check
cargo test -- --test-threads=1
cargo clippy --all-features --all-targets -- -D warnings
cargo check --examples --tests
```

Gotcha when re-pinning the patch block: `cargo update -p iroh ...` kept crates.io iroh 1.0.2 and reported the patches unused; `cargo update -p iroh --precise 1.3.2` moved it. When the lock holds two iroh entries (crates.io and the fork), name the old one: `cargo update -p iroh@<old> --precise <new>`. Update only the crates you name (`cargo update -p`), and revert any unrelated lock churn.

Whenever iroh-util moves, re-pin the iroh-docs fork's own `[patch.crates-io]` to it and run that fork's full suite (`cargo test --all-features`, 118 tests including `tests/sync.rs`): iroh-docs exercises the whole stack.

Aster validation after updating pins, from the Aster repository root:

```bash
cargo fmt --check
cargo clippy --manifest-path bindings/python/rust/Cargo.toml -- -D warnings
cargo test -p aster_transport_core
cargo test -p aster --all-features
uv run maturin develop -m bindings/python/rust/Cargo.toml
uv run pytest tests/python/ -v --timeout=30
```

Tag only after the Aster consumer trial passes against these exact commits (tags cannot be moved):

```bash
git tag -a aster-iroh-util-v<published version> -m "Aster iroh-util v<published version>" <release commit>
git push origin aster-iroh-util-v<published version>
# re-pin only: git tag -a aster-iroh-util-v<current version>-p<n> -m "..." <re-pin commit>
```

Update `iroh-fork-manifest.toml` in the Aster repository (the only copy; `[forks.iroh-util]`), and this ledger's Current release section on `main` (merge upstream `main` first). Local path-override consumers (for example portal-sync) build whatever is checked out, so switch the local checkout to the released tag after publishing.

## History

### iroh-family upstream wave (2026-11-09), current

| Upstream base | Tag | Pinned commit | Branch |
|---|---|---|---|
| `v0.7.0` | `aster-iroh-util-v0.7.1001` | `bfef8edbbe599bc4ea892860ea5218cf4a006844` | `upgrade/iroh-util-v0.7.0` |

- **0.7.1001** (`bfef8edb`): port onto upstream v0.7.0. Stack: `168bc6f` patch block (iroh 1.4.2002 `9ba1447d`, noq 1.4.1001 `379f73677c`), `a180db4` row 24 witness tests (from `db2c4dc`), `4ef2dca` row 33 re-cut on `ConnId` (from `be02c3d`), `bfef8ed` version. Row 24's rewrite half is upstream (#5/#12/#18). Option E: upstream patch 0, n = 1. iroh-blobs 0.104.2001 requires it (iroh-blobs row 34); Aster builds `Options` from `Default` because upstream made it `#[non_exhaustive]`. Validation at the pinned commit: `cargo test --locked --all-features -- --test-threads=1` lib 28 and doc 1 passed; clippy `--all-features --all-targets -D warnings` clean.

### iroh 1.4.2002 and noq 1.4.1001 re-pin (2026-10-28)

| Upstream base | Tag | Pinned commit | Branch |
|---|---|---|---|
| `cpfix` @ `aef1db3` (unchanged) | `aster-iroh-util-v0.6.3-p7` | `b28211ec000f013d7ad49cc9dd2fb93518c1ec8f` | `upgrade/iroh-util-v0.6` |

- **`aster-iroh-util-v0.6.3-p7`** (`b28211ec`): re-pin only, to iroh `9ba1447d` (Aster iroh 1.4.2002) and noq `379f73677c` (Aster noq 1.4.1001). Only Cargo.toml's `[patch.crates-io]` and Cargo.lock change (iroh, iroh-base, iroh-dns, iroh-relay to 1.4.2002; noq, noq-proto, noq-udp to 1.4.1001); the crate is not republished. Validation: `cargo test --locked --all-features`, 18 passed; clippy `--all-targets --all-features -D warnings` clean. The earlier commit `3467b0c1` (iroh `34ad5b11`, noq 1.4.3) was superseded by `b28211ec` before tagging, because noq's version moved from 1.4.3 to 1.4.1001 under the fork version rule (option E); nothing at it was tagged or published.

### iroh 1.4.2001 re-pin (2026-10-25)

| Upstream base | Tag | Pinned commit | Branch |
|---|---|---|---|
| `cpfix` @ `aef1db3` (unchanged) | `aster-iroh-util-v0.6.3-p6` | `502558d27d8ff0fc6645bac9c69c2bea159e3c34` | `upgrade/iroh-util-v0.6` |
| `cpfix` @ `aef1db3` (unchanged) | `aster-iroh-util-v0.6.3-p5` (pinned the superseded iroh 1.4.1, never published) | `16a93b49b482c4e7874ca1e91cae72f2cfb2616c` | `upgrade/iroh-util-v0.6` |

- **`aster-iroh-util-v0.6.3-p6`** (`502558d2`): re-pin only, to iroh `0739c259` (`aster-iroh-v1.4.2001`), the same iroh source as `aster-iroh-util-v0.6.3-p5` published under the fork version rule; the iroh 1.4.1 commit `aster-iroh-util-v0.6.3-p5` pinned was tagged but never published. Suite re-run at the new commit, all passed.
- **`aster-iroh-util-v0.6.3-p5`** (`16a93b49`): re-pin only, to iroh `68b684ea` (`aster-iroh-v1.4.1`: the port onto upstream 1.4.1 and row 44, the net report no longer waiting 3 s on an IPv6 path that drops UDP). Only Cargo.toml's `[patch.crates-io]` and Cargo.lock change; the crate is not republished. Validation: `cargo test --locked --all-features`, 18 passed.

### noq 1.4.1 and 1.4.2 re-pins (2026-10-22)

| Upstream base | Tag | Pinned commit | Branch |
|---|---|---|---|
| `cpfix` @ `aef1db3` (unchanged) | `aster-iroh-util-v0.6.3-p4` | `c9b91fce2eb8c1a30cfb27b24888f1b0ecaab556` | `upgrade/iroh-util-v0.6` |
| `cpfix` @ `aef1db3` (unchanged) | `aster-iroh-util-v0.6.3-p3` | `8c7947b638f1b5b9524b7b4aaa7c48855cf0e3b2` | `upgrade/iroh-util-v0.6` |

- **0.6.3-p3** (`8c7947b6`): re-pin only, to iroh `41ec78b1` (1.4.0-p1) and noq `13dcb5b7` (1.4.1).
- **0.6.3-p4** (`c9b91fce`): re-pin only, to iroh `91a92aa0` (1.4.0-p2) and noq `a3c70b7a` (1.4.2, incremental BBRv3 packet history); the lock follows with no other moves. Both touch only Cargo.toml's `[patch.crates-io]` and Cargo.lock; the crate stays 0.6.3 and is not republished. Validation on p4: `cargo test --locked --all-features` lib 18; clippy `--locked --all-features --all-targets -D warnings` and fmt clean.

### 1.4.0 wave (2026-10-21)

| Upstream base | Tag | Pinned commit | Branch |
|---|---|---|---|
| `cpfix` @ `aef1db3` (unchanged) | `aster-iroh-util-v0.6.3-p2` | `a90ea88d46b3178a5f1a7d4c3e28a34a2ad80aa4` | `upgrade/iroh-util-v0.6` |

- **2026-10-21, 0.6.3-p2** (`a90ea88d`, tag `aster-iroh-util-v0.6.3-p2`): re-pin only. No new upstream release (the base is still upstream `cpfix`), so no port. The patch block moves iroh, iroh-base and iroh-relay to `261ad623` (iroh 1.4.0, `upgrade/iroh-v1.4.0`) and noq, noq-udp and noq-proto to `535bd884` (noq 1.4.0, `upgrade/noq-v1.4.0`); the lock follows (rustls 0.23.46, rustls-webpki 0.103.16, lru 0.18.5). The commit touches only Cargo.toml's `[patch.crates-io]` and Cargo.lock. No source change; the crate stays 0.6.3 and is not republished. Validation: `cargo test --locked --all-features` lib 18; clippy `--locked --all-features --all-targets -D warnings` and fmt clean.

### 1.3.0 wave (2026-10-03 onward)

- **0.6.3** (`6c42d23`, tag `aster-iroh-util-v0.6.3`): row 33, `ConnectionPool::retire`, from the 2026-10-15 restart investigation.
- **2026-10-18, 0.6.3-p1** (`eb760160`, tag `aster-iroh-util-v0.6.3-p1`): the fork had no patch block and locked iroh and noq 1.0.0 from crates.io (review finding TM-7, found by `fork_compat.py --check`). It now pins iroh, iroh-base and iroh-relay to `a4a9a76b` and noq, noq-udp and noq-proto to `e75f3a55`, and the lock follows iroh 1.3.2's transitive moves (netwatch and portmapper 0.19.3, `n0-dns-resolver` in place of hickory). No source change; the crate stays 0.6.3 and the publisher skips it as identical. Validation: lib 18, clippy `--all-features --all-targets -D warnings`, `cargo fmt --check` and `check --examples --tests` clean.
- The iroh-docs fork's 0.101.5-p2 (2026-10-16) added iroh-util `6c42d238` (0.6.3) to its own patch block; its lock had previously taken 0.6.0 from crates.io.

### 1.0.1 wave (2026-07-02 onward)

| Upstream base | Tag | Pinned commit | Branch |
|---|---|---|---|
| `cpfix` @ `aef1db3` (upstream PR #5, over `iroh-util-v0.6.0`) | `aster-iroh-util-v0.6.2` | `afed9e6d07e41450fe1970e794782bab7df9c2f4` | `upgrade/iroh-util-v0.6` |

- **2026-08-24..26 (portal-sync cutover campaign), remediated 2026-08-27:** 0.6.1 and 0.6.2 (row 24) were committed directly on `upgrade/iroh-util-v0.6`, with no topic branch and no `-x` trailers. `test/connection-pool-witnesses` (`c6ca9970a3`, on upstream `cpfix` `aef1db3`) was reconstructed afterwards; it compiled and passed the fix's regression tests before push. Its trailers point at the upgrade-branch release commits, the opposite of the normal flow, because the releases existed first. Released tags and registry artifacts were not touched.
- **2026-08-25:** promoted from a checksum-verified bridge to a fork. 0.6.2 packages upstream's single-actor rewrite plus the portal-sync witness tests and supersedes the published-but-never-consumed 0.6.1 independent patch.
- Local checkout at the end of the cycle: `aster-iroh-util-v0.6.2`.

iroh-util was not an Aster fork before 2026-08-25, so it has no rows in the 1.0 wave or the rc.1 port.
