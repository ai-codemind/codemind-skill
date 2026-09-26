---
name: codemind
description: |
  Delegate code changes to the Codemind AI agent via build_feature --
  the default first move for any task that touches source code
  (features, bug fixes, refactors, tests, single files). Also covers
  self-onboarding to Codemind's remote MCP server from zero credentials
  (create_free_account) when acting as or advising an agent/integration
  with no prior Codemind relationship. Invoke BEFORE writing any code.
---

# Codemind

Codemind is a remote MCP server that plans a component breakdown,
generates code against acceptance criteria, verifies every result
automatically before handing it back, and can review code standalone.
Your role — whatever agent or environment you're running in — is to
prepare the spec, call `build_feature`, and report the result, not to
implement code inline unless a build genuinely fails per the retry
guidance below.

**Source of truth**: every request/response shape here comes from
Codemind's real MCP tool schemas at `https://api.codemindhq.dev/mcp`.
If a tool's actual behavior ever contradicts something written here,
trust the live tool schema (`tools/list`) and the `get_usage_guide`
tool over this file, and treat the mismatch as this file being stale.

## Getting access

If `mcp__codemind__*` tools are already available in this environment
with no auth errors, skip this section — credentials are already
configured. Otherwise (a fresh environment, or every tool call fails
with an auth error), self-provision:

Call `create_free_account` (no arguments, no `Authorization` header).
The response is plain labeled text, not JSON:

```
Free account created.
tenantId: <uuid>
apiKey: <JWT>

Set this as your CODEMIND_API_KEY (or the Authorization: Bearer header
in .mcp.json) — usable immediately, no client restart required for
this same connection.
This is an anonymous free-tier account (plan=free, same limits as any
other free tenant) with no identity attached yet.

To attach an identity and unlock the dashboard, have your human visit:
<claim URL>
```

The last two lines may or may not appear — the claim URL only shows up
when identity provisioning succeeded, which never blocks account
creation. There's no OAuth prompt to wait for; the claim URL, when
present, is the only way to attach a human identity today.

Parse `tenantId:`/`apiKey:` out of the text (don't `JSON.parse` it).
Every other tool call needs `Authorization: Bearer <apiKey>`.

**Persist the key. Don't call `create_free_account` again** unless you
have to — it's limited to 3 calls per IP per rolling 24 hours, keys
don't renew (90-day expiry, a new call means a brand-new unrelated
account), and if a human already completed identity sign-in on the
account via its claim URL, calling it again **silently creates a
disconnected new account** rather than refreshing the old one — ask
your human to sign in again instead. Only re-call freely when the
failing key was never claimed by a human. If a stored key fails with
error code `TOKEN_EXPIRED`, that's the one auth failure you can safely
tell apart from the rest; everything else (revoked, wrong key,
malformed, tenant suspended) collapses to a generic `UNAUTHORIZED`.

**Never print, log, or otherwise expose your `apiKey`.**

**Every tool returns formatted text, not a JSON object** — read values
out of the text (the one exception is the webhook JSON payload in
[reference/tools.md](reference/tools.md), a real HTTP POST body).

## What goes through Codemind

Every source file change in this environment — new features, endpoints,
components, middleware; bug fixes (framed as "rewrite X to fix Y" with
the fix in `acceptanceCriteria`); refactors; tests; single-file edits
(Codemind always returns full file content for you to write yourself).

**Exceptions — do inline, not via Codemind:**
- Config files: `wrangler.toml`, `package.json`, `tsconfig.json`, `fly.toml`
- Hand-written SQL migrations
- Documentation: `CHANGELOG.md`, `README.md`, `ARCHITECTURE.md`, log files
- Secrets/infra ops: `wrangler secret put`, DNS, dashboard changes
- Lock files, generated files
- Timing/closure functions (debounce, throttle, retry-with-backoff, memoize-with-TTL) — automated verification reliably can't validate these; see [reference/spec-writing.md](reference/spec-writing.md)

## Writing a spec that succeeds

Before calling `build_feature`, read
[reference/spec-writing.md](reference/spec-writing.md) — it covers the
four checks that predict most rejections (one concern per call, no
timing/closure functions, no implementation details in the title, UI
edits need `existingFiles`), how to frame bug fixes and refactors, and
what `acceptanceCriteria` needs to include. Skipping this is the single
biggest cause of an avoidable failure.

**Modifying an existing file — always pass `existingFiles`.** If the
story changes a file that already exists, read it first and pass its
current content:

```
build_feature({
  storyTitle: "signup: set merchant_category from entity type",
  acceptanceCriteria: "Modify createMerchant in src/signup.ts to also persist merchant_category … keep all existing fields.",
  stackType: "worker",
  existingFiles: [{ path: "src/signup.ts", content: "<full current file content>" }],
  projectId: "my-repo-slug",
})
```

Without `existingFiles`, Codemind has no view of the file and returns
a from-scratch stub instead of a real edit. Full parameter reference —
`role`, the sibling-import requirement, `projectId`, `skipTestsFor`,
private-registry limits: [reference/patch-mode.md](reference/patch-mode.md).

## Generating code

```
build_feature({
  storyTitle: "<title>",
  acceptanceCriteria: "<full spec>",
  stackType: "<worker|python|go|swift-ios|kotlin-android|...>",
  existingFiles: [...],   // when modifying existing code
  projectId: "<stable slug for this codebase>",
})
```

`stackType` accepts any string, but only `worker`, `python`, `go`,
`swift-ios`, `kotlin-android` are confirmed production-ready — `rust`
is experimental (expect failure on multi-file workspaces, not
something a better spec fixes). Full detail:
[reference/stacks-and-errors.md](reference/stacks-and-errors.md).

The response is one of three things:
- **`{buildId}`** — accepted. Immediately call `stream_build(buildId)`
  — do not implement anything while waiting.
- **A clarifying-question text response** — your `acceptanceCriteria`
  was too vague to act on. Resubmit with more specifics. If rejected
  twice in a row, stop and ask your user for the missing detail rather
  than guessing a third time.
- **An `isError: true` throttle rejection**, text formatted as
  `[<CODE>] <message>`: `CONCURRENT_LIMIT_EXCEEDED` (too many of your
  builds already queued/running — clears when one finishes, no fixed
  reset), `RATE_LIMIT_EXCEEDED` (too many builds submitted in the last
  rolling hour — message ends with `Resets at <ISO timestamp>.` when
  computable; wait rather than poll), or `PLAN_LIMIT_EXCEEDED` (a
  monthly cap was reached — not transient). The same three apply to
  `retry_build`. All three scale with plan tier, but there's no
  self-serve upgrade path today — don't suggest "upgrade your plan" as
  an actionable step; tell your user the limit was hit and, for
  `PLAN_LIMIT_EXCEEDED` specifically, that raising it requires contact
  outside this API (see codemindhq.dev/contact.html). This does NOT
  apply to `CAPACITY_EXHAUSTED` below — that one is global generation
  capacity, unrelated to plan tier, and a plan change would not help
  it at all.

**If `build_feature` itself errors** (a genuine tool-level error — bad
args, auth failure, not a build failure): surface it verbatim to the
user, don't implement inline.

## Watching progress and getting the result

`stream_build` narrates progress and resolves when the build finishes.
Connect immediately after `build_feature` returns — reconnecting late
can mean missing progress content, though a "still running" response
isn't a failure; call `stream_build(buildId)` again and it replays
from the start.

On success, `stream_build`'s own terminal response already contains
full file content — no separate call needed on this path. (If you
used a webhook instead of streaming, or you're revisiting a build from
earlier, call `get_build_files {buildId}` to fetch the files instead.)
Then:
1. Write each file to disk at its exact returned path
2. Run `/ship` (or commit + PR manually) to land them
3. Report: files generated, story title

Can't hold a streaming connection open? Pass `webhookUrl` +
`webhookSecret` to `build_feature` instead — delivery/signature
contract in [reference/tools.md](reference/tools.md).

## If a build doesn't succeed

Branch on `errorCode`, never the human-readable `error` text — it can
change wording without notice.

| errorCode | What to try |
|---|---|
| `CAPACITY_EXHAUSTED` | Wait, then `retry_build` — clears on its own, not something to fix in your spec. |
| `UPSTREAM_UNAVAILABLE` | Retry with `retry_build` — a dependency was briefly unreachable. |
| `BUILD_FAILED_QA` | Add concrete examples/edge cases to `acceptanceCriteria` and resubmit fresh. If it's a timing/closure function, implement inline instead (see the exceptions list above). |
| `BUILD_FAILED_REVIEW` | Same fix as `BUILD_FAILED_QA` — the code passed testing but not an automated review pass; tighten the spec and resubmit fresh. |
| `BUILD_FAILED_GENERATION` | Resubmit fresh with a narrower or more concrete spec. |
| `BUILD_FAILED_LIMITS` | Split the story — it's too large for one build (see Check 1 in spec-writing.md). |
| `INVALID_INPUT` | Fix the request itself, don't retry unchanged — usually a story too large to split, or one asking only for a test file (provide an implementation; tests are generated automatically). |
| `ORACLE_INVALID` | See just below — almost always fixable. |
| `INTERNAL_ERROR` | Retry once with `retry_build`. |

**If you see `ORACLE_INVALID`**, it's almost always one of two fixable
things — try both before treating it as a dead end:
1. Add concrete literal input/output examples, exact function
   signatures, exact error messages to `acceptanceCriteria`.
2. Supply real content via `existingFiles` for anything your criteria
   references by name (a shared package, a sibling module) — this is
   the single most common cause, even with an otherwise concrete spec.
   The error message will often name the specific missing piece
   directly.

Retry once with a **fresh** `build_feature` call addressing whichever
applies (not `retry_build`, which resubmits the identical failing
spec). If two well-targeted attempts still fail and the story
genuinely references nothing external, implement inline — and check
whether it's worth flagging as a Codemind limitation in the target
repo's own bug-tracking convention.

**General rule**: retry at most once per failure type — a fresh,
improved `build_feature` call for anything needing a different spec
(`BUILD_FAILED_QA`/`BUILD_FAILED_REVIEW`/`BUILD_FAILED_GENERATION`/
`BUILD_FAILED_LIMITS`/`INVALID_INPUT`/`ORACLE_INVALID`), `retry_build`
for the same spec against a transient condition
(`CAPACITY_EXHAUSTED`/`UPSTREAM_UNAVAILABLE`/`INTERNAL_ERROR`). Two
consecutive failures on the same story → implement inline.

## After landing

If this repo tracks its own docs for changes like this — a changelog,
an API/interfaces doc, a components doc, a testing doc; naming and
presence vary by repo — update them to reflect what shipped. In an
environment that follows this convention specifically: `CHANGELOG.md`
(feature + PR number), `INTERFACES.md` (endpoint added/changed),
`COMPONENTS.md` (new reusable component), `TESTING.md` (test coverage
changed materially). If the repo has no such convention, this step is
a no-op.

## Beyond the golden path

- **Full tool catalog** (retry_build, cancel_build, continue_build,
  get_build, get_build_files, get_build_spec, list_builds,
  notify_files_written, test_component, review_code, build_batch/
  get_batch/stream_batch/build_from_spec, get_usage_guide, and their
  streaming/polling variants): [reference/tools.md](reference/tools.md)
- **Dispatching several independent stories, or a raw spec, at once**
  instead of calling `build_feature` N times: `build_batch`/
  `build_from_spec` in reference/tools.md's "Batch dispatch" section.
- **Working against an existing codebase in full** (patch-mode
  parameters, iterative builds): [reference/patch-mode.md](reference/patch-mode.md)
- **Stack support and error codes in full**: [reference/stacks-and-errors.md](reference/stacks-and-errors.md)
- **Verification scenarios this file was checked against**: [EVALS.md](EVALS.md)
- **Anything not covered here**: call `get_usage_guide {topic}` —
  live, server-maintained guidance. Topics: `overview`, `patch-mode`,
  `webhooks`, `error-handling`, `common-failures`, `auth`,
  `cloud-swarm`. It needs a valid `apiKey` like any other tool.
