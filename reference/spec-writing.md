# Writing a `build_feature` spec that won't get rejected

## STOP — four mandatory pre-submission checks

Before calling `build_feature`, verify all four. Any failure → fix first.

**Check 1 — One concern only.**
If the `storyTitle` or `acceptanceCriteria` describes 2+ independent behaviors (connected by "and", "+", ","), **STOP. Split into separate `build_feature` calls first.**
- Bad: `"Sprint 1: chat completions, session management, and quota tracking"` → 3 builds
- Bad: `"JWT auth middleware + resolver ownership checks"` → 2 builds
- Bad: `"CI/CD pipeline + HMAC validation + unit tests"` → 3 builds (CI/CD is config anyway — skip Codemind)
- Bad: `"EPIC-11: API key authentication for all endpoints"` → epic-level, decompose first (or use `build_batch`/`build_from_spec` — see `tools.md`'s "Batch dispatch" section — instead of decomposing by hand)

**Check 2 — No timing/closure functions.**
Debounce, throttle, retry-with-backoff, once, memoize-with-TTL — these involve timer internals or closure state that Codemind's automated verification reliably can't validate (a known hard category, not something a better spec fixes). **Implement these inline instead of submitting them**, then optionally get the same verification rigor with `test_component`/`stream_test` against your own code.

**Check 3 — No technology names in the title.**
If the title contains `jwt callback`, `AES-GCM`, `D1 cache`, `NextAuth`, `localStorage`, `useEffect`, `useState`, or any other implementation detail → rewrite as what the *user or caller experiences*.

**Bad (describes implementation → INVALID_INPUT):**
```
storyTitle: "refactor(dashboard): provision backend API key in NextAuth jwt callback, drop AES-GCM D1 cache"
```

**Good (describes behavior):**
```
storyTitle: "auth: surface backend API key to authenticated dashboard sessions"
acceptanceCriteria: |
  Authenticated users must be able to make backend API calls from the dashboard
  without a separate key-entry step. The system must:
  - Include a valid backend API key in the session available to client components
  - Return 401 if the key is missing or invalid at the backend
  - Test: authenticated request succeeds; unauthenticated request returns 401
```

**Check 4 — UI single-file edits need `existingFiles`.**
For React/TSX changes to an existing component (e.g. adding a delete button, a new tab, a modal), pass the current file content via `existingFiles` (see `patch-mode.md`) and frame the story as what the user sees/does — not what component state changes.
- Bad: `"Add delete button to Documents table"` → no criteria, no `existingFiles` → `ORACLE_INVALID`/from-scratch stub
- Good: `"documents: user can delete a document from the table"` + criteria specifying the DELETE call, success/error behavior, table refresh, and the current file passed via `existingFiles`

## Framing bug fixes and refactors

**Bad fix framing**: `storyTitle: "fix null crash in rate limiter"` (names the bug, not the required behavior)

**Good fix framing**:
```
storyTitle: "fix: rate limiter crashes on null KV response"
acceptanceCriteria: |
  The rate limiter in src/lib/rate-limiter.ts currently throws an unhandled
  TypeError when KV.get() returns null (simulated by KV outage). It must:
  - Catch null/undefined KV response and throw RateLimiterUnavailable
  - NOT return 200 or silently pass the request through
  - Test: unit test covering the null-KV path returns 503
  Current behavior: unhandled TypeError propagates to the route, returns 500 with stack trace.
```
Pass the current file content via `existingFiles`.

**Refactor framing** — describe the replacement, not the removal:
```
storyTitle: "refactor: replace direct fetch() calls in engine-client with retry wrapper"
acceptanceCriteria: |
  src/lib/engine-client.ts makes bare fetch() calls with no retry. Rewrite to:
  - Use the existing fetchWithRetry() helper from lib/http.ts
  - Preserve all current method signatures and return types
  - All existing tests must continue to pass
```

## Expand `acceptanceCriteria` — always include

A one-word or one-phrase `storyTitle` ("string utilities", "helpers", "auth module") without `acceptanceCriteria` **will always fail `INVALID_INPUT`/`ORACLE_INVALID`**. Always specify at least one concrete test case before submitting:

- Happy path behavior
- Error/edge cases (null, missing, timeout, 4xx, 5xx)
- Constraints specific to the target repo (error shape, tenant scoping, auth, rate limits — check that repo's own CLAUDE.md/RULES.md)
- Test signal: "at least one test must verify X behavior"
- For bug fixes: current broken behavior + expected fixed behavior
- For API changes: exact request/response shapes with field names and types
- **Every external dependency the criteria references (a shared package, a sibling module, a type) must have its real content supplied via `existingFiles`** — the test Codemind generates will try to import it for real, and an unresolvable import is one of the most common `ORACLE_INVALID` causes even with an otherwise maximally concrete spec.
