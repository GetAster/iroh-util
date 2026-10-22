# Aster patch ledger: iroh-util

This repository is Aster's fork of [n0-computer/iroh-util](https://github.com/n0-computer/iroh-util). Aster carries the patches listed below on top of upstream. Upstream maintainers are welcome to incorporate any of them, in whole or in part, at their discretion.

## Branch and tag model

- `main` is upstream `main` plus one commit: the Aster README preamble and this ledger. Nothing is built from `main`. It is refreshed with `git merge upstream/main` and is never reset or force-pushed. Ledger edits are commits on `main`, never on `upgrade/*`.
- `upgrade/<repo>-v<upstream version>` (for example `upgrade/iroh-v1.4.0`, `upgrade/iroh-blobs-v0.103.1`) is the patched integration line: the upstream release plus the patches in this ledger. A new upstream base always gets a new branch, cut from the upstream release tag; Aster-only fixes and re-pins append to the current branch. Per-minor names (`upgrade/iroh-blobs-v0.103`, `upgrade/salvo-v1.0`, …) are historical. Consumers pin commits on it. Published `upgrade/*` branches and tags are never rewritten; a failed attempt is superseded by new commits and a new version.
- Topic branches carry one change each, are named for the change (for example `feat/read-into`, `security/plumtree-bounds`), are based on the upstream release tag (or, for a patch that builds on another, on that prerequisite patch's commit, which its row names), and are cherry-picked with `git cherry-pick -x` onto the upgrade branch, so every upgrade-branch commit names its source. A topic branch a row names is not rewritten; re-cutting it onto a new upstream base takes a versioned name, `<topic>-v<upstream version>`.
- Version: the upstream version if Aster's registry has never served that version or a higher one in the series; otherwise the next patch above Aster's highest published version. A re-pin without a code change keeps the version and takes the tag suffix `-p<n>`.
- Tags: `aster-<repo>-v<published version>` (re-pins `aster-<repo>-v<published version>-p<n>`), created only after the Aster consumer trial passes against those exact commits, because tags cannot be moved.
- The fork manifest is `iroh-fork-manifest.toml` in the Aster repository (the only copy).
- Aster does not submit patches upstream; upstream maintainers are welcome to take any patch here.

Historical and current tag schemes in this repository:

- `aster-iroh-util-v<version>`: `aster-iroh-util-v0.6.1`, `-v0.6.2`, `-v0.6.3`.
- `aster-iroh-util-v<version>-p<n>`, a later commit at the same crate version: `aster-iroh-util-v0.6.3-p1`, `-v0.6.3-p2`, `-v0.6.3-p3`, `-v0.6.3-p4` (patch-block re-pins with no source change).

## Current release

| Field | Value |
|---|---|
| Upstream base | `cpfix` @ `aef1db3` (upstream PR n0-computer/iroh-util#5, over `iroh-util-v0.6.0`). Not an upstream release tag: the base is upstream's unreleased single-actor `ConnectionPool` rewrite. |
| Integration branch | `upgrade/iroh-util-v0.6` (a historical per-minor name; the next upstream base gets `upgrade/iroh-util-v<upstream version>`) |
| Tag | `aster-iroh-util-v0.6.3-p4` |
| Pinned commit | `c9b91fce2eb8c1a30cfb27b24888f1b0ecaab556` |
| Crate version | 0.6.3 |
| Source repository | `https://forge.emrul.dev/Aster/iroh-util.git` |

The fork's own `[patch.crates-io]` pins iroh, iroh-base and iroh-relay to `91a92aa0` (Aster iroh 1.4.0-p2) and noq, noq-udp and noq-proto to `a3c70b7a` (Aster noq 1.4.2), the stack Aster links it against.

Bootstrap pin line for source-tree builds (consumers install a versioned Aster SDK release instead and must not copy this):

```toml
iroh-util = { git = "https://forge.emrul.dev/Aster/iroh-util.git", rev = "c9b91fce2eb8c1a30cfb27b24888f1b0ecaab556" }
```

Where this fork lives: iroh-util was a checksum-verified registry bridge until 2026-08-25, when it was promoted to a fork for the connection-pool deadlock (row 24). It first lived at `github.com/aster-rpc/iroh-util`, until the Forgejo org repository existed. It then moved to Forgejo because it was the last fork not there, and the only one whose fetch could be refused by someone else: GitHub rate-limits anonymous git per source address, and when the Forge host crossed that limit the release stopped here, reporting "could not read Username for 'https://github.com'", which reads like a private repository rather than a throttled public one. The Forgejo repository is the same history (same commit ids, same tags), and the Aster publisher (`publish-native-stack.py`) checks out the pinned commit and fails unless `git rev-parse HEAD` matches, so provenance rests on the hash, not the hostname. Forgejo push-mirrors it to `github.com/getaster/iroh-util`, the public copy.

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

Order across the iroh family: noq, then iroh, then the satellites in the Aster publisher's order: this repository (its patch block names noq and iroh by revision), iroh-gossip, iroh-blobs (row 34 needs `ConnectionPool::retire` from row 33), iroh-docs.

Before porting, check upstream:

1. If a released iroh-util contains PR #5, base on that release and keep only the witness tests from row 24.
2. If upstream `main`'s #12/#18 rewrite (`ConnId`, `UnusedSet`) has been released, re-express row 33 on it as described in its conflict notes.

```bash
git fetch --all --tags --prune
git push origin <upstream-tag>          # mirror the upstream release tag (e.g. iroh-util-v0.7.0) to the forge
git switch -c upgrade/iroh-util-v<upstream version> <upstream-tag>
# re-point [patch.crates-io] at the new iroh / noq fork revs (iroh, iroh-base, iroh-relay; noq, noq-udp, noq-proto), commit
git cherry-pick -x c6ca9970a3     # row 24 witness tests (as needed on the new base)
git cherry-pick -x 5b67790        # row 33
# version bump, only if the version rule requires it (Aster's registry has already
# served <upstream version> or higher in the series, as it has 0.6.1..0.6.3):
# set the crate to the next patch above Aster's highest published, then
git commit -am "chore: release <published version>"
git push origin upgrade/iroh-util-v<upstream version>
# do not tag yet: tag after the consumer trial (below)
```

The current base is not an upstream release tag (upstream `cpfix` `aef1db3`, see Current release). Once upstream releases a version containing PR #5, cut the new branch from that release tag as above.

Prefer `cherry-pick -x`. Published `upgrade/*` branches and tags are never rewritten; a failed attempt is superseded by new commits and a new version.

An Aster-only fix on the current base is a topic branch, cherry-picked `-x` onto the current upgrade branch, followed by a version-bump commit to the next patch above Aster's highest published version (as 0.6.3 was cut for row 33). A re-pin without a code change is a commit on the current upgrade branch with no version bump, tagged `aster-iroh-util-v<current version>-p<n>` after the consumer trial (as `aster-iroh-util-v0.6.3-p1` was).

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

### noq 1.4.1 and 1.4.2 re-pins (2026-10-22), current

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
