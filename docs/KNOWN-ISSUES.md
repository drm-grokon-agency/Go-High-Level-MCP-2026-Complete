# Known Issues

Diagnostic findings from live investigation. Each entry separates observed behavior
from mechanism and states its evidence level. "Runtime" = observed against the
GoHighLevel API; "Source" = confirmed in this repository's code.

---

## Trigger create is broken: the write does an `update` on a document that does not exist yet

**Recorded:** 2026-07-07
**Status:** Open - not fixed.
**Where:** trigger write path - `create_trigger` in `src/tools/triggers-tools.ts`, and
`create_workflow` when called with a trigger (it routes through the same trigger-write code).

**Observed behavior (runtime):**
The trigger write performs a Firestore **update** against the trigger document. On a
create, that document does not exist yet, so GHL's backend returns `NOT_FOUND`, surfaced
to the client as HTTP `500`. Net result: the code can **edit an existing trigger** but can
**never create a new one**.

**Impact:** `create_workflow`-with-a-trigger goes through this same path, and this path is
what was **hanging the entire MCP server** on that call.

**Evidence:**
- Runtime: `NOT_FOUND` on the `500` returned by the trigger write.
- Source: `create_trigger` handler exists in `src/tools/triggers-tools.ts`; the
  `create_workflow`-with-trigger flow uses the same trigger-write code (confirmed by inspection).
- The Firestore update-vs-create detail is GHL server-side; it is inferred from the
  `NOT_FOUND` signature, not visible in this repo's source.

**To confirm before fixing:**
- Capture the raw `500` body on a trigger *create* to confirm the `NOT_FOUND` origin and
  identify the exact upstream write call the handler builds.
- Pin down *how* the `NOT_FOUND`/`500` manifests as a multi-minute hang rather than a fast
  error (a fast 500 alone would not hang the server - look for a retry loop or an
  un-timed-out await on the trigger write).

**Fix direction (unverified):**
- Trigger create needs a create/set (write a new document), not an update against a
  nonexistent one.
- Add a timeout / fail-fast on the trigger write so a `500`/`NOT_FOUND` cannot hang the
  server - this addresses the `create_workflow` hang independently of the create fix.