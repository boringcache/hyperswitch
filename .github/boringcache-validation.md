# BoringCache validation for issue #14195

This branch tests shared sccache storage for the MSRV compilation job on
GitHub-hosted runners. The validation keeps the upstream source at
`02ba5ea0b4cc0ad8debea4cefd28a7de5916d9db` and adds only the BoringCache
configuration and validation workflows.

The workflow uses GitHub OIDC. It does not store a BoringCache token. The first
job can publish compiler results; the second job runs on a clean runner with
restore-only access.

## Result

[Run 34979495882](https://github.com/boringcache/hyperswitch/actions/runs/34979495882)
completed successfully on September 15, 2026.

| Measurement | Publish job | Clean-runner restore job |
| --- | ---: | ---: |
| `cargo check --features "release"` | 12m19s | 3m40s |
| Complete job | 13m00s | 4m24s |
| Executed cache lookups | 3,332 | 3,327 |
| Cache hits | 275 | 3,325 |
| Cache misses | 3,052 | 2 |
| Hit rate | 8.27% | 99.94% |
| Cache read errors | 0 | 0 |

The restore reduced the compilation step by 8m39s (70.2%) and the complete job
by 8m36s (66.2%). BoringCache recorded 586 MB read and 581 MB written across
both jobs. It measured a 2 ms average cache-hit service time and a 3 ms p95
interval average.

The restore job had seven remote missed lookups in BoringCache's HTTP evidence;
sccache classified two of them as compiler-cache misses. sccache attempted to
write the two newly compiled results and reported two write errors because the
job had restore-only access. The job had no cache read errors and completed
successfully.

GitHub reported that the pinned upstream `arduino/setup-protoc` action still
targets Node.js 20. GitHub ran it on Node.js 24 for this test. This warning is
independent of BoringCache.

This result measures one unchanged-source publish/restore pair. It does not yet
measure rolling source changes, merge-queue concurrency, or fork pull-request
access.
