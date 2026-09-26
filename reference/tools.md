# Codemind tool reference

This covers every tool except the three golden-path ones documented in
`SKILL.md` itself: `create_free_account`, `build_feature`, `stream_build`.

## Contents
- Build lifecycle: retry_build, cancel_build, continue_build, get_build, get_build_files, get_build_spec, list_builds
- notify_files_written
- Standalone testing: test_component, stream_test, get_test
- Standalone review: review_code, stream_review, get_review
- Batch dispatch (Cloud Swarm): build_batch, get_batch, stream_batch, build_from_spec
- get_usage_guide
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

**continue_build** `{buildId, correctedAcceptanceCriteria}` — steers a
`running` build without cancelling it: replaces the acceptance
criteria for every component that hasn't started generating yet.
Components already generating finish against the ORIGINAL criteria —
this doesn't retroactively fix them. If every remaining component has
already started, the correction is accepted but has no effect (you get
a success message either way, not an error — there's no way to tell
"applied" from "too late" apart from the response text). A `buildId`
not in `running` status returns a non-error "nothing to steer" message
naming the actual status. Use this when you realize your
`acceptanceCriteria` was wrong shortly after submitting, instead of
`cancel_build` + a fresh `build_feature` call.

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

## Batch dispatch (Cloud Swarm)

Use these instead of calling `build_feature` N times yourself when you
already have several independent stories, or a raw spec to decompose.
They exist for volume — a single build is still just `build_feature` +
`stream_build`.

**build_batch** `{items: [{repoUrl, storyTitle, acceptanceCriteria, stackType, existingFiles?, skipTestsFor?}, ...], idempotencyKey?, webhookUrl?, webhookSecret?}` —
1 to 50 items per call. `repoUrl` is informational only (not used to
fetch code — same as `build_feature`, code still comes from
`existingFiles`). Returns one `batchId` immediately; items beyond a
small burst cap are queued and dispatched automatically on a later
drain tick, not dropped. `idempotencyKey`: a repeat call with the same
key returns the original `batchId` instead of dispatching everything a
second time — pass it if your client might retry the call after a
timeout, omit it if you always want a genuinely new batch.
`webhookUrl`/`webhookSecret` here fire **once**, when every item in the
batch reaches a terminal state (see below) — separate from each item's
own per-build webhook, which this tool does not accept (poll
`get_batch`/`stream_batch`, or each item's own `buildId`, for
per-build progress instead).

**Known gaps, not yet fixed** — codemind#1642: items submitted through
`build_batch`/`build_from_spec` skip the spec-clarity gate that a
direct `build_feature` call gets, so an under-specified item fails
`BUILD_FAILED_QA` later instead of getting an immediate
clarifying-question response. codemind#1643: in a narrow crash window,
`get_batch`/`stream_batch` can misreport a still-`queued` item's real
status until a 24h timeout resolves it. Neither blocks normal use.

**get_batch** `{batchId}` — aggregate status:
`{batchId, status, total, queued, running, completed, failed, items: [{repoUrl, storyTitle, status, buildId?, errorCode?}, ...]}`.
`status` is `running` while any item is `queued`/`running`, else
`completed` (zero failures) or `completed_with_failures`. A `batchId`
that doesn't exist, or belongs to a different tenant, both return the
identical `isError: true` "batch not found" text — you can't
distinguish a foreign batch from a nonexistent one.

**stream_batch** `{batchId}` — polls the same aggregate status
periodically and emits a progress notification each tick; closes once
`status` is `completed` or `completed_with_failures`. It only covers
items already dispatched at the moment you call it — items the queue
drains later won't retroactively show up in that same call, re-call
`stream_batch`/`get_batch` later to check on those. Prefer `get_batch`
for a single point-in-time snapshot.

**A failed individual item has no batch-specific retry** — call
`retry_build {buildId}` directly on that item's `buildId` (from
`get_batch`). This mints a **new** `buildId` not linked back to the
batch: `get_batch`/`stream_batch` for the original `batchId` will keep
reporting that item as `failed` even after the retry succeeds, and the
aggregate webhook (already fired once) will not refire. Track a
retried item's own outcome via `stream_build`/`get_build` on the new
`buildId`.

**build_from_spec** `{specText, repos: [{repoUrl, hint?}, ...], maxStories?, dryRun?, webhookUrl?, webhookSecret?}` —
give it raw PRD/plan text plus the repos it's allowed to target;
Codemind LLM-decomposes it into stories (each assigned to exactly one
of your declared `repos` — the model can never invent a `repoUrl`
outside that list) and dispatches them the same way `build_batch`
does. `maxStories` defaults to 20, hard ceiling 50. Pass `dryRun: true`
on a first call to review the decomposed story list before committing
anything — a dry run makes zero writes, so it's free to retry with
adjusted `specText`/`repos`. To submit an EDITED version of a dry run's
output, call `build_batch` directly with your edited story list — a
second `build_from_spec` call re-decomposes `specText` from scratch and
won't reflect any edits you made to the dry run's output.

## get_usage_guide

`{topic?}` — live, server-maintained guidance for calling agents;
`topic` defaults to `overview` if omitted. Valid topics: `overview`,
`patch-mode`, `webhooks`, `error-handling`, `common-failures`, `auth`,
`cloud-swarm`. Call this — not this Skill's own files — whenever you
need guidance on something this Skill doesn't cover, or when a tool's
actual behavior seems to contradict what's written here.

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
