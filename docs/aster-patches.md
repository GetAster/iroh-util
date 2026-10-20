# Aster patch ledger: iroh-util

This repository is Aster's fork of [n0-computer/iroh-util](https://github.com/n0-computer/iroh-util). Aster carries the patches listed below on top of upstream. Upstream maintainers are welcome to incorporate any of them, in whole or in part, at their discretion.

## Branch and tag model

- `main` is the front door: upstream's README with an Aster preamble, plus this ledger. Nothing is built from `main`.
- `upgrade/<version>` is the patched integration line for one upstream release wave: the upstream release plus the patches in this ledger. Consumers pin it by commit.
- Topic branches carry one change each and are named for what the change is, for example `feat/read-into` or `security/plumtree-bounds`. Each is based on the upstream release tag and is cherry-picked with `git cherry-pick -x` onto the upgrade branch, so every upgrade-branch commit names its source.
- Release tags mark the commits that are released and pinned. Published tags are never moved or rewritten. Historical tags in this repository use several schemes (listed below); new tags use `aster-<repo>-v<version>`, where `<version>` is the published version of the repository's main package (for example `aster-salvo-v1.0.4`, `aster-fory-v1.7.6-aster.1`).
- `main` is refreshed by merging upstream `main` into it. It is not force-pushed.

Tag schemes used in this repository:

- `aster-iroh-util-v<version>`: `aster-iroh-util-v0.6.1`, `-v0.6.2`, `-v0.6.3`.
- `aster-iroh-util-v<version>-p<n>`, a later commit at the same crate version: `aster-iroh-util-v0.6.3-p1` (a patch-block re-pin with no source change).

## Current release

| Field | Value |
|---|---|
| Upstream base | `cpfix` @ `aef1db3` (upstream PR n0-computer/iroh-util#5, over `iroh-util-v0.6.0`). Not an upstream release tag: the base is upstream's unreleased single-actor `ConnectionPool` rewrite. |
| Integration branch | `upgrade/iroh-util-v0.6` |
| Tag | `aster-iroh-util-v0.6.3-p1` |
| Pinned commit | `eb76016080ea037464a7de71196b6de2999736b2` |
| Crate version | 0.6.3 |
| Source repository | `https://forge.emrul.dev/Aster/iroh-util.git` |

The fork's own `[patch.crates-io]` pins iroh, iroh-base and iroh-relay to `a4a9a76b` (Aster iroh 1.3.2) and noq, noq-udp and noq-proto to `e75f3a55` (Aster noq 1.3.2), the stack Aster links it against.

Bootstrap pin line for source-tree builds (consumers install a versioned Aster SDK release instead and must not copy this):

```toml
iroh-util = { git = "https://forge.emrul.dev/Aster/iroh-util.git", rev = "eb76016080ea037464a7de71196b6de2999736b2" }
```

Where this fork lives: iroh-util was a checksum-verified registry bridge until 2026-08-25, when it was promoted to a fork for the connection-pool deadlock (row 24). It first lived at `github.com/aster-rpc/iroh-util`, until the Forgejo org repository existed. It then moved to Forgejo because it was the last fork not there, and the only one whose fetch could be refused by someone else: GitHub rate-limits anonymous git per source address, and when the Forge host crossed that limit the release stopped here, reporting "could not read Username for 'https://github.com'", which reads like a private repository rather than a throttled public one. The Forgejo repository is the same history (same commit ids, same tags), and the Aster publisher (`publish-native-stack.py`) checks out the pinned commit and fails unless `git rev-parse HEAD` matches, so provenance rests on the hash, not the hostname. `github.com/aster-rpc/iroh-util` remains as the upstream-facing copy.

## Patches

Row numbers are stable patch ids shared across Aster's iroh-family forks (iroh, iroh-blobs, iroh-docs, iroh-gossip, iroh-util, noq), so a commit message that says "row 33" resolves here.

| Id | Topic branch | What | Why / what in Aster depends on it | Commits (topic / upgrade) | Status | Conflict or porting notes |
|---|---|---|---|---|---|---|
| 24 | upstream `cpfix` (`aef1db3`) + `test/connection-pool-witnesses` (tip `c6ca9970a3`, on `aef1db3`; backfilled) | Adopts upstream's single-actor `ConnectionPool` rewrite (n0-computer/iroh-util PR #5), replacing the 0.6.0 design whose two blocking-send cycles deadlocked the pool at more than 100 concurrent `get_or_connect` calls (portal-sync's freeze-apply-window wedge). Adds three Aster witness tests: a 300-connect burst resolves under saturation; dropping the pool terminates the actor and closes held connections; slow-peer backlogs do not starve an unrelated peer. File: `src/connection_pool.rs`. | Every portal-sync blob fetch and docs sync dial goes through this pool; the deadlock silently froze whole daemons. | `aef1db3` + `c6ca9970a3` / `396759d` + `afed9e6`; released as 0.6.2 | carried | When a released iroh-util ≥ 0.6.x contains PR #5, this fork collapses back to carrying only the witness tests. The topic branch was reconstructed after release, so its `(cherry picked from …)` trailers point at the upgrade-branch commits (see History). |
| 33 | `fix/pool-retire-connection` (tip `5b67790`, on upstream `cpfix` `aef1db3`) | Adds `ConnectionPool::retire(&ConnectionRef)`: closes the connection with reason `retired` and forgets it, only while the pool still holds that same connection (matched by connect generation), so the next `get_or_connect` dials anew and a late or repeated retire never closes the replacement. Ready connections, close-watcher events and unused events carry the generation, so no event for a replaced connection touches its successor. `close(id)` is unchanged. File: `src/connection_pool.rs`. | Root cause 3 of the Aster restart/reconnect investigation (Aster repository, `docs/_internal/investigations/2026-10-15-restart-reconnect.md`): the iroh-blobs downloader (iroh-blobs fork row 34, which requires 0.6.3) must drop its pooled connection to a provider that restarted on a new port without killing a fresh replacement. | `5b67790` / `10c23d8`; released as 0.6.3 at `6c42d23` | carried | **Aster's own; no upstream equivalent (checked 2026-10-15): n0-computer/iroh-util issues/PRs #1–#19 (open #2 per-peer backlog; draft #13 adopts incoming connections and marks the older one `superseded`, a different trigger; merged-unreleased #12 generation-tagged events, #17, #18 slot reservation) and n0-computer/iroh-blobs #141, #185, #213, #224, #226, #227, #229.** Medium conflict risk against upstream `main`'s #12/#18 rewrite (`ConnId`, `UnusedSet`): on re-port, express `retire` as "remove `current` if its ConnId matches, close with `retired`" and drop the generation plumbing that #12 already provides. Test: `retire_dials_anew_and_spares_the_replacement` (the retired connection closes `LocallyClosed`; the next get dials a new one; a stale second retire leaves the replacement open and pooled); it fails if retire ignores the generation. lib 18 single-threaded, clippy `--all-targets --all-features -D warnings` and fmt clean. |

### Upstreamed or dropped

| Patch | Commit / tag | Status | When and why |
|---|---|---|---|
| Independent Aster fix for the 0.6.0 connection-pool deadlock (0.6.1) | `fc724bd`, tag `aster-iroh-util-v0.6.1` | dropped | Published to the registry but never consumed; superseded on 2026-08-25 by 0.6.2, which adopts upstream's PR #5 rewrite instead (row 24). The tag is preserved for the record. Do not consume or republish it. |

## Porting to a new upstream release

Order across the iroh family: noq, then iroh, then this repository (its patch block names both by revision), then iroh-blobs (row 34 needs `ConnectionPool::retire` from row 33), then iroh-docs.

Before porting, check upstream:

1. If a released iroh-util contains PR #5, base on that release and keep only the witness tests from row 24.
2. If upstream `main`'s #12/#18 rewrite (`ConnId`, `UnusedSet`) has been released, re-express row 33 on it as described in its conflict notes.

```bash
git fetch --all --tags --prune
git switch -c upgrade/iroh-util-<new-version> <new-upstream-base>
# re-point [patch.crates-io] at the new iroh / noq fork revs (iroh, iroh-base, iroh-relay; noq, noq-udp, noq-proto)
git cherry-pick -x c6ca9970a3     # row 24 witness tests (as needed on the new base)
git cherry-pick -x 5b67790        # row 33
git tag -a aster-iroh-util-v<published version> -m "Aster iroh-util v<published version>"
git push origin upgrade/iroh-util-<new-version> aster-iroh-util-v<published version>
```

Prefer `cherry-pick -x`. Do not rewrite a published `upgrade/*` branch or tag unless you are deliberately replacing a failed attempt.

Validate in this repository:

```bash
cargo fmt --check
cargo test -- --test-threads=1
cargo clippy --all-features --all-targets -- -D warnings
cargo check --examples --tests
```

Gotcha when re-pinning the patch block: `cargo update -p iroh ...` kept crates.io iroh 1.0.2 and reported the patches unused; `cargo update -p iroh --precise 1.3.2` moved it.

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

Update the fork manifest (canonical `docs/aster-fork-manifest.toml` in the Aster iroh fork, `[forks.iroh-util]`; synced copy `iroh-fork-manifest.toml` in the Aster repository) and this ledger's Current release section. Local path-override consumers (for example portal-sync) build whatever is checked out, so switch the local checkout to the released tag after publishing.

## History

### 1.3.0 wave (2026-10-03 onward), current

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
