# Evaluations

These are manual verification scenarios (Codemind Agent Skills have no
built-in automated eval runner) — walk each one with a fresh agent that
has read only this Skill's files, against the real
`https://api.codemindhq.dev/mcp` endpoint.

1. **Cold start.** Given no prior Codemind account, ask the agent to
   build a one-file Cloudflare Worker `/health` endpoint. Expect: it
   calls `create_free_account` exactly once, then `build_feature` →
   `stream_build` → `get_build_files`, with no human touching
   credentials at any point.

2. **QA failure.** Force or find a build that ends `BUILD_FAILED_QA`.
   Expect: the agent's next action branches on `errorCode`, considers
   `retry_build` or a more specific `acceptanceCriteria`, and does not
   try to parse or "fix" the English `error` string itself.

3. **Existing-file edit.** Ask the agent to add a field to a file it's
   told already exists. Expect: it uses `existingFiles`+`role`, not a
   blind full-file regeneration request.

4. **Vague spec, twice.** Give the agent a deliberately vague feature
   request. Expect: after 2 spec-gate clarifying-question rejections,
   it stops and asks its user for the missing detail rather than
   guessing a 3rd time.

5. **Free-tier exhausted.** Simulate (or catch, if it happens for real)
   `create_free_account` responding with `isError: true` because the
   3-per-24h IP limit was already hit (the exact wording isn't
   documented — don't assert a specific error code for this case).
   Expect: the agent tells its user it can't self-provision further,
   rather than retry-looping.

## Acceptance criterion

An agent with zero prior context, given only this Skill and either raw
HTTP or a declarative MCP client, reaches a working file set/PR for a
simple one-file Worker feature without a human handling credentials,
calls `create_free_account` at most once ever, and correctly branches
on at least the `BUILD_FAILED_QA` and rate-limit failure modes rather
than guessing.
