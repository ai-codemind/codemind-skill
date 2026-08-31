# Codemind tool reference

This covers every tool except the three golden-path ones documented in
`SKILL.md` itself: `create_free_account`, `build_feature`, `stream_build`.

## Contents
- Build lifecycle: retry_build, cancel_build, get_build, get_build_files, get_build_spec, list_builds
- notify_files_written
- Standalone testing: test_component, stream_test, get_test
- Standalone review: review_code, stream_review, get_review
- Outbound webhooks (webhookUrl/webhookSecret)

## Build lifecycle

**retry_build** `{buildId}` — only works on a `failed` build; resubmits
the identical original spec (`storyTitle`, `acceptanceCriteria`,
`stackType`, `existingFiles`, `skipTestsFor`). Does NOT carry over the
original build's `webhookUrl`/`webhookSecret` — the retried build has
no webhook configured unless you separately submit a fresh
`build_feature` call with new webhook params instead of using
`retry_build`. Not-found or wrong-status → `isError: true` text.

**cancel_build** `{buildId}` — hard-stops an in-flight build. Already-terminal
builds return a non-error "nothing to cancel" message, not an error.

**get_build** `{buildId}` — point-in-time status/result, no streaming.

**get_build_files** `{buildId}` — generated files, formatted as fenced
code blocks. Works once status is `completed` OR `failed` (a failed
build may still have partial files from components that finished
before the failure).

**get_build_spec** `{buildId}` — returns the original
`{storyTitle, acceptanceCriteria, stackType, existingFiles?, skipTestsFor?}`
you submitted, so you can inspect a build without retrying it.

**list_builds** `{limit?, offset?}` — no product/status filtering on
this API version.

## notify_files_written

`{projectId, files: [{path, content}]}` — indexes file content for a
project (up to 10 files / 700KB total) without submitting a build.
Useful to prime context before a later `build_feature` call that
passes the same `projectId`.

## Standalone testing

**test_component** `{stackType, component: {filePath, exportName, signature, purpose?, supportingTypes?}, code, acceptanceCriteria, contextFiles?}` —
verifies code YOU supply against acceptance criteria; generates
nothing. Returns a `testId`.

**stream_test** `{testId}` — same streaming pattern as `stream_build`,
resolves with pass/fail + the synthesized verification test.

**get_test** `{testId}` — point-in-time poll, same shape as `get_build`.

## Standalone review

**review_code** `{files?, diff?, context?, conventions?, relatedFiles?}` —
at least one of `files`/`diff` required. Two independent LLM passes
(general correctness/security + a dedicated silent-failure pass);
findings merge and pass through an anti-hallucination filter before
you see them. Returns a `reviewId`.

**stream_review** `{reviewId}` — resolves with `verdict`
(`approve`/`request-changes`) + findings. A result may carry
`degradedPasses` (which pass didn't complete) — do not read a review
missing this warning as full coverage if it's present.

**get_review** `{reviewId}` — point-in-time poll, same shape as `get_build`.

## Outbound webhooks

Pass both `webhookUrl` (https-only, ≤2048 chars) and `webhookSecret`
(16–256 chars) to `build_feature` to get a POST instead of streaming.
Both are required together or omit both.

On the build reaching `completed` or `failed`, your `webhookUrl`
receives:

```json
{
  "event": "build.completed | build.failed",
  "buildId": "string",
  "tenantId": "string",
  "status": "completed | failed",
  "storyTitle": "string",
  "stackType": "string",
  "errorCode": "string | null",
  "error": "string | null",
  "filesTotal": "number | null",
  "sentAt": "number (unix seconds)"
}
```

Verify authenticity: compute `HMAC-SHA256(webhookSecret, <exact request body bytes>)`,
compare (constant-time) against the hex digest in the
`x-codemind-v2-signature` header after stripping its `sha256=` prefix.
Cross-check `x-codemind-v2-build-id` against the payload's own
`buildId`, and treat a `sentAt` more than a few minutes old as stale.

Delivery is at-least-once — your endpoint must be idempotent against
duplicate deliveries of the same `buildId`+`event`.
