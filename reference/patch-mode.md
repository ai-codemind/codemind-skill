# Working against an existing codebase

## existingFiles

Pass `existingFiles: [{path, content, role?}]` on `build_feature` to
patch specific files instead of generating a whole new codebase.

- `role: "target"` — forces this path to become its own patchable
  component, even if your `acceptanceCriteria` doesn't name it
  explicitly. Use this specifically when a prior failed build's result
  named the path under `unmatchedExistingFiles` — that means the
  planner saw the file but didn't treat it as something to patch.
- `role: "context"` — this file is supplied only to resolve some
  *other* file's import, not to be patched itself. It exempts this
  file's own imports from the sibling-import check below — unless your
  `acceptanceCriteria` quotes this exact path verbatim, in which case
  it's treated like a normal entry and the exemption doesn't apply.
- Omitted `role` — **not** the same as `context`: this file's own
  relative imports are always required to be supplied, identically to
  `target`. `role` only adds a signal about whether the planner treats
  the path as a guaranteed component; it never removes the import
  requirement.
- **A test-file path (`*.test.ts`, `*_test.go`, `test_*.py`, `*Test.kt`,
  etc.) cannot be an `existingFiles` entry at all** — Codemind always
  synthesizes its own verification test per component, so submitting
  one is rejected outright, regardless of `role`.

## The sibling-import check

Before submission, Codemind scans every relative import (`./x`, `../x`,
Python `from .x import`) inside your non-`context` `existingFiles`
entries. If an import's target isn't also in `existingFiles`, the call
is rejected with a message telling you which file to add (or to mark
the importing file `role: "context"` if it's not actually a component
you want built). This only applies to import-syntax stacks (TS/JS,
Python) — Swift/Kotlin have no equivalent check.

## projectId

Pass the same `projectId` across multiple `build_feature` calls against
the same codebase (combined with `existingFiles`) to get codebase-context
enrichment, and every build automatically reindexes its own output for
the next call. Use this for "keep iterating on the same project," not
one-off builds.

## skipTestsFor

`skipTestsFor: ["path/one.ts", "path/two.ts"]` — up to 20 repo-relative
paths where Codemind should not author test code. You own the
unverified-code risk for those specific files. Other files in the same
build still get normal test coverage, and if *every* component in the
build ends up test-skipped, the build still fails loudly (Codemind
refuses to ship a build with zero test signal).

## npmrc

Pass raw `.npmrc` content as `npmrc` on `build_feature` to authenticate
private npm registries inside the build sandbox (e.g. GitHub Packages).
Not persisted — sandbox-scoped to this one build only.
