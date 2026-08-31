# Stacks and error codes

## stackType values

`stackType` is a free-text field — the API does not validate it
against a fixed list, so a typo or unsupported value will not
necessarily fail fast with a clear error. Stick to known-good values.

**Confirmed production-ready:**

| stackType | Runtime | QA method |
|---|---|---|
| `worker` | TypeScript / Cloudflare Workers | Real test execution (dynamic workers backend) |
| `python` | Python 3 | Real test execution (pytest, sandboxed) |
| `go` | Go modules | Real test execution (go test, sandboxed) |
| `swift-ios` | Swift | Real test execution (sandboxed) |
| `kotlin-android` | Kotlin | Real test execution (sandboxed) |

**Experimental:** `rust` — reliably plans and generates, but QA
reliably fails on a known multi-crate-workspace vs. single-crate-harness
mismatch. Expect `BUILD_FAILED_QA`/`ORACLE_INVALID`, not a working
result — this is a known limitation, not something a better
`acceptanceCriteria` will fix.

**Retired — do not use:** `pages`, `expo`, `infra`, `design`.

## errorCode reference

| errorCode | Meaning |
|---|---|
| `CAPACITY_EXHAUSTED` | No LLM capacity available right now |
| `BUILD_FAILED_LIMITS` | Output exceeded size/token limits — narrow scope |
| `BUILD_FAILED_QA` | Generated code failed automated testing |
| `BUILD_FAILED_REVIEW` | Generated code failed an LLM code review pass |
| `BUILD_FAILED_GENERATION` | The generator failed to produce valid output |
| `INVALID_INPUT` | Request was malformed |
| `UPSTREAM_UNAVAILABLE` | A dependency was unreachable — usually transient |
| `INTERNAL_ERROR` | Unexpected server-side failure |
| `ORACLE_INVALID` | Codemind's own test harness couldn't validate the result — not your bug |

`error` is always a human-readable string with no model names, account
IDs, or provider details. Branch your logic on `errorCode`, never on
this string.
