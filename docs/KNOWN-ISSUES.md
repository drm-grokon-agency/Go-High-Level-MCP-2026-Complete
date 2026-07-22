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

---

## `ghl_get_workflow_executions` calls a route that does not exist on the public API

**Recorded:** 2026-07-21
**Status:** Open - not fixed.
**Where:** `getWorkflowExecutions` in `src/tools/workflow-tools.ts`.

**Observed behavior (runtime):**
The handler issues `GET /workflows/{workflowId}/executions` against the public API host
(`services.leadconnectorhq.com`, via `makeRequest`). That path returns HTTP `404` with the
Express route-missing body (`"Cannot GET /workflows/.../executions"`). The route does not
exist on the public API, so the tool can never return execution/enrollment data. Its
`contactId` / `status` / `startDate` / `endDate` parameters are an untested input schema,
not a working server-side filter.

**Impact:** Any feature built on this primitive (e.g. a per-contact automation-history
fan-out) cannot work as written - it would loop a call that 404s.

**Evidence:**
- Runtime: `GET services.leadconnectorhq.com/workflows/{wf}/executions?locationId=...` ->
  `404 {"message":"Cannot GET /workflows/.../executions...","error":"Not Found","statusCode":404}`
  (probed live with a valid PIT + `Version: 2023-02-21`, location `KWEEmHNX0lTGtgifAALH`).
- Runtime: six sibling public paths also `404` route-missing: `/contacts/{id}/workflows`,
  `/contacts/{id}/workflow`, `/workflows/{wf}/enrollments`, `/workflows/{wf}/enrollment-history`,
  `/workflows/{wf}/contacts`, `/workflows/executions`.
- Runtime: the internal definition service `backend.leadconnectorhq.com/workflow` authenticates
  (minted Firebase id_token + `Bearer {PIT}` + `token-id`) and serves `/{loc}/list`,
  `/{loc}/{wf}`, `/{loc}/{wf}/history` (200), but returns `404 {"msg":"Not found"}` for
  `/{loc}/{wf}/executions|enrollments|stats|contacts` - execution/enrollment data is not on
  that service either.
- Source: `getWorkflowExecutions` builds the path via the public `makeRequest` (confirmed by inspection).

**Not yet located:** the endpoint that actually serves per-contact enrollment/execution
history. It exists (the workflow builder "Execution logs" / "Enrollment history" tabs render
it) but its address has not been captured - the builder is a cross-origin iframe and its XHR
was not observable via available tooling. Next step: capture that request from the browser
DevTools Network panel.

**Fix direction (unverified):**
- Repoint `getWorkflowExecutions` at the real endpoint once its host/path is captured, reusing
  the internal auth already proven for `backend.leadconnectorhq.com` (mint Firebase id_token;
  send `Bearer {PIT}` + `token-id`).
- Until then, treat the tool as non-functional so callers do not mistake its schema for a working filter.