---
name: codemind
description: |
  Generate, test, and review code via the Codemind MCP server -- plan a
  component breakdown, run it through an execution-verified QA ladder, and
  get back working files or a PR. Use when the user asks to build a
  feature, fix a bug, write tests for existing code, or get an automated
  code review, and wants an AI agent to delegate the actual code
  generation/verification to a remote service rather than writing it inline.
---

# Codemind

Codemind is a remote MCP server that generates, tests, and reviews code
against acceptance criteria, verifying every result in a real sandbox
before handing it back.

**Source of truth**: every request/response shape in this Skill comes
from Codemind's v2 API at `https://api.codemindhq.dev/mcp`. If you find
other Codemind documentation mentioning a `repoUrl` or `product` field,
or a base URL other than `api.codemindhq.dev`, that documentation
describes a different, decommissioned API version — do not mix its
shapes with this Skill's.

## Getting access

You need an API key before calling any tool except `create_free_account`.

Call `create_free_account` (no arguments) with **no** `Authorization`
header. The response is plain labeled text, not JSON — it looks like:

```
Free account created.
tenantId: <uuid>
apiKey: <JWT>

Set this as your CODEMIND_API_KEY (or the Authorization: Bearer header
in .mcp.json) — usable immediately, no client restart required for
this same connection.
```

Parse the `tenantId:` and `apiKey:` lines out of the text — don't
`JSON.parse` the response. Every other tool call needs
`Authorization: Bearer <apiKey>` on the request. If you're running
inside an MCP client with config-file-based auth (e.g. `.mcp.json`),
set `apiKey` as the value your config expands into that header — per
the tool's own response, this takes effect immediately on the same
connection, no reconnect needed.

**Persist the key. Do not call `create_free_account` again** in this
environment — it's limited to 3 calls per IP per rolling 24 hours, and
there is no renewal for the key once it expires (90 days): a new key
means a brand-new, unrelated account, not a refresh of the old one. If
`create_free_account` itself is rate-limited, or a stored key stops
working, tell your user rather than retrying — only a human can
provision a paid account at that point.

**Never print, log, or otherwise expose your `apiKey`.**

**Every tool below returns formatted text, not a JSON object** — read
values out of the text rather than expecting a structured payload
(the one exception is the webhook JSON payload described near the end
of [reference/tools.md](reference/tools.md), which is a real HTTP
POST body, not a tool-call result).

## Generating code

Call `build_feature`:

```json
{
  "storyTitle": "Add a /health endpoint",
  "acceptanceCriteria": "GET /health returns 200 with JSON {status: 'ok'}",
  "stackType": "worker"
}
```

`stackType` accepts any string (it isn't validated against a fixed
list), but only these are confirmed production-ready: `worker`,
`python`, `go`, `swift-ios`, `kotlin-android`. `rust` is explicitly
experimental — it reliably plans and generates but fails QA on a known
harness mismatch, so expect `BUILD_FAILED_QA`/`ORACLE_INVALID` rather
than a working result. Other slugs you may see referenced elsewhere
(`pages`, `expo`, `infra`, `design`) are retired — don't use them.
See [reference/stacks-and-errors.md](reference/stacks-and-errors.md)
for detail.

The response is either `{buildId}` or a clarifying-question text
response (the spec-clarity gate rejected your `acceptanceCriteria` as
too vague). If rejected, resubmit with a more specific
`acceptanceCriteria`. If rejected twice in a row, stop and ask your
user for the missing detail instead of guessing a third time.

## Watching progress and getting the result

Call `stream_build {buildId}` **immediately** after `build_feature`
returns — don't do other work first. (Token-level progress content
isn't replayed to a late subscriber; connecting even a few seconds
late can mean missing all of it.)

`stream_build` narrates progress and resolves when the build finishes.
If it instead returns a "still running" message, that is not a
failure — call `stream_build {buildId}` again; it always replays full
progress history from the start, so you won't miss anything by
reconnecting.

On success, call `get_build_files {buildId}` for the generated files.

On failure, branch on `errorCode` — never parse the human-readable
`error` message:

| errorCode | Meaning | Retry with `retry_build`? |
|---|---|---|
| `CAPACITY_EXHAUSTED` | No LLM capacity available right now | Yes, likely transient |
| `UPSTREAM_UNAVAILABLE` | A dependency (sandbox, LLM provider) was unreachable | Yes, likely transient |
| `BUILD_FAILED_QA` | Generated code failed automated testing | Maybe — consider a more specific `acceptanceCriteria` first |
| `BUILD_FAILED_REVIEW` | Generated code failed an LLM code review pass | Maybe — same as above |
| `BUILD_FAILED_GENERATION` | The generator itself failed to produce valid output | Maybe |
| `BUILD_FAILED_LIMITS` | Output exceeded size/token limits | No — narrow the scope of `acceptanceCriteria` instead |
| `INVALID_INPUT` | Your request was malformed | No — fix the request |
| `ORACLE_INVALID` | Codemind's own test harness couldn't validate the result | No — this is not your bug to fix; report if it recurs |
| `INTERNAL_ERROR` | Unexpected server-side failure | Yes, once |

Can't hold a streaming connection open? Pass `webhookUrl` and
`webhookSecret` to `build_feature` instead of calling `stream_build` —
see [reference/tools.md](reference/tools.md) for the delivery and
signature-verification contract.

## Beyond the golden path

- **Full tool catalog** (retry_build, cancel_build, get_build,
  get_build_files, get_build_spec, list_builds, notify_files_written,
  test_component, review_code, and their streaming/polling variants):
  [reference/tools.md](reference/tools.md)
- **Working against an existing codebase** (patching specific files,
  iterative builds on the same project): [reference/patch-mode.md](reference/patch-mode.md)
- **Stack support and error codes in full**: [reference/stacks-and-errors.md](reference/stacks-and-errors.md)
- **Verification scenarios this Skill was checked against**: [EVALS.md](EVALS.md)
