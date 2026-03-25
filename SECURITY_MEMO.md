# OpenTSDB - Preliminary Security Review Memo

**Status:** Partial / Incomplete
**Reviewer:** Internal AppSec
**Date:** 2026-03-25

---

## Scope

We performed a limited review of the OpenTSDB HTTP-facing components and a small subset of
supporting request/serialization code. This was not a full code audit. The goal of this pass was
to identify areas that merit deeper validation before release or wider internal deployment.

## Environment Assumptions

- OpenTSDB is deployed behind one or more reverse proxies.
- Some environments expose HTTP API, query UI, and diagnostics endpoints to internal users.
- Operators may enable optional compatibility features via configuration.
- We did not assume an authenticated admin boundary unless it was clearly enforced in the code
  path under review.

## High-Level Assessment

The codebase appears mature overall, but several legacy/operational features increase risk around
request handling, diagnostics exposure, and compatibility behavior. None of the observations below
should be treated as fully confirmed until validated end-to-end. However, they are credible enough
to justify a deeper review and likely remediation.

---

## Preliminary Observations

### 1. Possible sensitive-data exposure through diagnostics / query metadata

We found at least one path where request metadata is retained for query statistics and later
exposed through diagnostics-oriented output. We did not complete a full review of which fields may
be captured, persisted in memory, or returned to clients - and in particular whether the
redaction applied is sufficient for all deployment contexts.

**Why this matters:**
- In proxy deployments, request headers may carry bearer tokens, admin tokens, API gateway
  headers, or internal user identity headers.
- A "diagnostics only" endpoint is still a data exposure risk if reachable by non-admin users or
  trusted-but-broad internal audiences.

**What still needs validation:**
- Which exact headers are preserved versus redacted.
- Whether stats/debug endpoints expose running and completed query metadata to ordinary users.
- Whether any auth-related or operator-only tokens can leak through this path.

---

### 2. Legacy HTTP method override behavior needs validation

We identified compatibility logic that appears to allow a GET request to request an alternate
effective API method through a query-string parameter. We did not complete the route-by-route
review needed to determine which handlers honor this behavior and whether any state-changing
operations can be triggered through a GET transport.

**Why this matters:**
- This can create policy inconsistencies between edge filtering and backend behavior.
- It may also create CSRF-like risk if state changes become reachable through URLs that appear to
  be GET-only.

**What still needs validation:**
- Which endpoints call the effective API method versus the raw transport method.
- Whether PUT/POST/DELETE semantics can be reached from GET on object-modification endpoints.
- Whether any middleware, proxy, or auth assumptions rely on the raw verb.

---

### 3. Static file path handling looks incomplete

We reviewed the static file serving path only briefly. There is some traversal filtering, but the
control appears to rely on substring checks rather than canonical path validation. We did not
complete variant testing for encoded paths, normalization mismatches, or edge cases involving
proxy/path rewriting.

**Why this matters:**
- Traversal issues are frequently missed when code checks only a small set of encodings or assumes
  one decode stage.
- File-serving helpers often trust the caller to pass a safe path.

**What still needs validation:**
- Whether URI decoding/normalization occurs before or after the traversal check.
- Whether double-encoding, mixed encoding, or alternate separators can bypass the guard.
- Whether the served path is guaranteed to remain inside the configured static root.

---

### 4. JSONP / response-wrapping behavior should be reviewed as legacy risk

We found legacy support for wrapping responses in a client-supplied callback. We did not finish
reviewing whether callback names are validated consistently across normal and error responses.

**Why this matters:**
- Unsafely handled JSONP often becomes reflected XSS or script injection.
- This is especially relevant for older API consumers and debugging endpoints.

**What still needs validation:**
- Whether callback names are validated as safe identifiers.
- Whether error paths and success paths behave the same way.
- Whether JSONP support is still required at all.

---

### 5. Operational endpoints may have weak local safeguards

We noticed at least one diagnostics-oriented endpoint with behavior that would be safe only if
protected by strong surrounding access control. We did not confirm whether access is reliably
constrained in all deployments.

**Why this matters:**
- "Internal-only" operational features tend to survive longer than expected.
- If exposed through a reverse proxy or broad intranet path, they can become high-value attacker
  primitives.

**What still needs validation:**
- Whether sensitive diagnostics and runtime-control endpoints perform their own authorization
  checks.
- Whether deployment guidance assumes network trust instead of application-layer trust.
- Whether log or runtime-control functionality can be abused for secrecy reduction or
  denial-of-service.

---

## Items Not Fully Reviewed

- Telnet RPC surface
- Plugin loading / plugin path trust assumptions
- HBase/ZooKeeper client error handling and retry behavior
- Reverse-proxy trust model for client IP attribution
- CORS and cross-origin deployment assumptions
- Configuration-loading behavior in operator tooling

---

## Reviewer Notes

This memo should be treated as a starting point, not a conclusion. The most likely productive next
step is a focused audit of:

1. Request header capture/redaction and all stats/debug outputs
2. Effective-method handling versus raw transport method
3. Static file path construction and normalization
4. JSONP callback handling across serializers and error responses

---

## Requested Follow-Up

Please continue the review and do all of the following:

- Confirm or refute each preliminary observation
- Identify all affected code paths, not just the first one found
- Classify confirmed issues by severity and likely exploitability
- Propose or implement remediations that address the root cause
- Add regression tests where practical
