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

Terse lookup table — see `SKILL.md`'s "If a build doesn't succeed"
section for the full guidance, including `ORACLE_INVALID`'s two-step
fix.

| errorCode | What it means | What to try |
|---|---|---|
| `CAPACITY_EXHAUSTED` | No generation capacity available right now | Wait, then `retry_build` |
| `UPSTREAM_UNAVAILABLE` | A dependency was briefly unreachable | `retry_build` |
| `INTERNAL_ERROR` | Unexpected server-side failure | `retry_build` once |
| `BUILD_FAILED_QA` | Generated code failed automated testing | Add concrete examples to `acceptanceCriteria`, resubmit fresh |
| `BUILD_FAILED_REVIEW` | Code passed testing but not an automated review pass | Same fix as `BUILD_FAILED_QA` |
| `BUILD_FAILED_GENERATION` | Code generation didn't produce valid output | Narrower/more concrete spec, resubmit fresh |
| `BUILD_FAILED_LIMITS` | Request too large for one build | Split the story |
| `INVALID_INPUT` | Request malformed, or asked only for a test file | Fix the request — don't retry unchanged |
| `ORACLE_INVALID` | Couldn't validate the result against your criteria | Add concrete examples, or supply real content via `existingFiles` for anything referenced by name |

`error` is always a human-readable string with no model names, account
IDs, or provider details. Branch your logic on `errorCode`, never on
this string.
