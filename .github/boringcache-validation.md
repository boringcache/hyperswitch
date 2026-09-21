# Hyperswitch cache validation

This fork evaluates BoringCache against the build-cache problem reported in
[Hyperswitch #14195](https://github.com/juspay/hyperswitch/issues/14195). The
issue remains open. The original GitHub-cache repair in
[#14196](https://github.com/juspay/hyperswitch/pull/14196) closed without a
merge. Hyperswitch now has a separate open implementation in
[#14253](https://github.com/juspay/hyperswitch/pull/14253) that streams a local
sccache directory to and from S3.

The current question is therefore not whether Hyperswitch needs persistent
cache storage. It does. The question is whether BoringCache can replace enough
of its custom S3, credential, retention and reporting work to justify a pilot.

## Qualification

Hyperswitch uses ephemeral Kubernetes ARC runners. The configuration described
in #14195 lost local sccache state when each pod exited, while GitHub cache
storage was reported at 9.56 GiB of its 10 GB allowance. The issue recorded
47-minute test and 25-minute MSRV baselines.

PR #14253 addresses much of that pain without BoringCache. Its current design:

- restores and saves the complete local sccache directory as a streamed S3
  tarball;
- uses PR-specific keys with a shared-cache fallback;
- permits restore and save failures with `continue-on-error`;
- depends on runner-provided S3 configuration and IAM access;
- skips the affected jobs for pull requests from forks; and
- reports a 37m08s to 5m19s change for its `build-hyperswitch` job on the
  upstream self-hosted pool.

That result is not a control for this validation. The upstream run used a
different runner pool, source revision and cache implementation. BoringCache's
fork cannot access Hyperswitch's private runners or S3 bucket.

## Integration

The primary validation uses the released BoringCache One v1.31.0 commit
`9ca311f9b247835b3cc87639321c79bac8018145`. GitHub OIDC supplies the cache
capability. No static BoringCache token is stored in the fork.

The full-Cargo lane uses the v1.31 job lifecycle: the action restores Cargo
dependencies and `target`, starts a read-only or writable sccache proxy, the
workflow runs the ordinary Cargo command, and the action publishes in its post
step. This keeps restore, build and publish timings separate. The compiler-only
lane uses the standalone sccache adapter.

Both lanes use:

- `ubuntu-24.04` x86-64 GitHub-hosted runners;
- Rust 1.98.1, selected by `stable 2 weeks ago` during the run;
- sccache 0.17.0 and mold;
- `CARGO_BUILD_JOBS=2` and `CARGO_INCREMENTAL=0`;
- `cargo build --package router --bin router`;
- seed source `c1d7b1ee0fb2a9efcff9a5b631bf24f59f9aaa44`; and
- rolling source `0defa221a3939d47f4e3259f4142ff8c8c42205b`.

The rolling source keeps the seed's `Cargo.lock`. It changes eight files in
Hyperswitch workspace crates. The unchanged and rolling jobs restore only; the
seed job publishes.

## Primary v1.31 measurements

The primary full-Cargo evidence is
[run 35648109211](https://github.com/boringcache/hyperswitch/actions/runs/35648109211).
All three jobs passed.

| Case | Whole job | Restore phase | Cargo command | Cache result |
| --- | ---: | ---: | ---: | --- |
| Fresh-tag seed | 20m34s | 0.2s, 0 bytes | 18m41s | Miss; published after the build |
| Unchanged source | 2m01s | 65.7s, 3,369,775,043 bytes | 8s | Target and compiler cache hit |
| Rolling source | 15m29s | 65.6s, 3,369,775,043 bytes | 13m22s | Target and compiler cache hit |

The restored set represents four entries: Cargo registry cache, registry index,
Git database and `target`. The CLI reported 26.72 GB of logical target data and
11,700 target files. The target entry accounted for 2,804,642,752 transferred
bytes. The other three entries accounted for the remaining 565,132,291 bytes.

The seed publish phase took 73.8s and transferred 701,626,459 changed bytes
across four entries. The fresh tags guaranteed a restore miss, but not an empty
workspace-wide content store. Earlier Hyperswitch validations had already
published shared blobs, so this is a deduplicated publication measurement, not
a globally cold upload measurement.

The unchanged build made three non-cacheable compiler calls and executed no
compilations. The rolling build made fourteen compiler requests, with ten Rust
misses and no compiler hits; the restored Cargo target avoided the other work.
Those ten read-only write denials appear as sccache write errors. Cache read
errors and timeouts were zero, so the denials are not backend failures.

The compiler-only evidence is
[run 35644615176](https://github.com/boringcache/hyperswitch/actions/runs/35644615176).
All three jobs passed.

| Case | Whole job | Cargo command | Rust hits / misses |
| --- | ---: | ---: | ---: |
| Fresh-tag seed | 36m23s | 35m33s | 0 / 936 |
| Unchanged source | 5m58s | 5m19s | 935 / 1 |
| Rolling source | 15m05s | 14m16s | 925 / 11 |

The large difference between the two cold command times, despite matching
source, command, toolchain and runner class, demonstrates material hosted-runner
variance. The single full-Cargo and compiler-only samples must not be treated as
a controlled performance comparison. The warm results establish successful
reuse and identify where time remains; they do not establish a production
speedup over Hyperswitch's S3 implementation.

The compiler-only rolling job recorded a 98.82% Rust hit rate but still spent
14m16s in Cargo. Hit rate alone does not describe the changed-source result.
Workspace recompilation and linking remain material after dependency objects
hit the compiler cache.

## Retained attempts

[Run 35644615220](https://github.com/boringcache/hyperswitch/actions/runs/35644615220)
used v1.31 with the older wrapped-command lifecycle. Its seed and unchanged jobs
passed. The rolling job restored all four entries, then GitHub sent the runner a
shutdown signal. There was no cache or build error before shutdown. This run is
not substituted into the primary series.

The wrapped run also confirmed that v1.31 restored the target entry in about 64
seconds. The earlier v1.30.4 full-Cargo run
[35624335081](https://github.com/boringcache/hyperswitch/actions/runs/35624335081)
took about 169 seconds to restore its four entries. These are single runs with
different source pairs and cache contents; they support another repeated
measurement, not a release-regression claim.

## Product fit

Hyperswitch remains a qualified prospect, but the commercial case is now
operational consolidation rather than basic cache persistence.

BoringCache can plausibly replace the custom S3 restore/save scripts with:

- OIDC-scoped publish and restore capabilities, including restore-only access
  for untrusted jobs;
- one managed or BYOC layer for Cargo dependencies, target state and compiler
  objects;
- content-level deduplication instead of a complete PR-scoped tarball upload;
- cache-session evidence, transfer measurements and retained cache identity;
  and
- fail-open operation in production while keeping strict failure checks in a
  validation lane.

The risks are also concrete:

- PR #14253 already gives Hyperswitch a working S3-backed design and measured
  warm results.
- The full-Cargo restore transferred 3.37 GB on every fresh runner in this
  sample.
- Rolling source still spent 13m22s in Cargo after a successful target restore.
- This validation did not run on Hyperswitch's private ARM64/AMD64 ARC pools.
- It did not inject a mid-build outage, so it does not prove fail-open behavior
  under the failure that motivated the Spice.ai qualification.
- It did not measure a trusted rolling publish, so the expected incremental
  upload advantage remains unmeasured here.

## Required next comparison

Run a same-runner comparison on Hyperswitch's ARC pool at one fixed source and
toolchain:

1. PR #14253's S3 tarball design.
2. BoringCache compiler-cache only.
3. BoringCache full Cargo lifecycle.

Use at least five independent seeds and five fresh-runner unchanged and rolling
jobs per lane. Record whole-job, restore, Cargo and publish duration; transferred
and logical bytes; compiler hits and misses; runner architecture; cache errors;
and storage requests. Add one controlled cache outage after the build begins and
verify that the build result survives while publication failure remains visible.
Test a trusted writer plus a restore-only fork pull request before proposing an
upstream workflow change.

Until that comparison exists, outreach should ask whether removing the custom
S3 cache lifecycle, credentials and reporting work is valuable enough to run
the pilot. It should not claim that BoringCache is faster than PR #14253.
