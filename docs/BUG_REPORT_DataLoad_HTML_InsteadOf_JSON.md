# Bug Report: Incentive Comp Tool — "Failed to load data: Unexpected token '<', \"<html> <h\"... is not valid JSON"

**Date:** 2026-02-23  
**Reporter:** (Generated from user report and codebase analysis)  
**Related repositories:** Incentive Comp Tool (this app), [Shopify/data-portal-mcp](https://github.com/Shopify/data-portal-mcp) (backend / data portal — *repo returned 404; assumed internal*)

---

## Summary

When users click **⚡ Load Live Data** in the Incentive Compensation Tool, the app can fail with:

```text
Failed to load data: Unexpected token '<', "<html> <h"... is not valid JSON
```

The application expects **JSON** from the data warehouse API but receives an **HTML** response (e.g. an error or redirect page). The JSON parse then throws, and the user sees the above message.

### Who must act

**The data-portal-mcp (or Quick data-layer backend) must be updated** to fix this. The server is returning HTML where the client expects JSON; that can only be fixed on the backend. Updating the Incentive Comp Tool report or UI does not resolve the underlying failure—it only documents it and improves the error message. **Action:** Implement the changes in the "Required changes" section below in data-portal-mcp (or the service that serves `quick.dw` query responses).

---

## Environment

- **App:** Incentive Compensation Tool (Quick site)
- **Live URL:** https://incentive-compensation-tool.quick.shopify.io
- **Client stack:** Static `index.html` + `/client/quick.js` (Quick.js runtime), `quick.dw.querySync()` for BigQuery
- **Backend / data layer:** Presumed to be served by the Quick platform and/or **data-portal-mcp** (or related services) that proxy/execute BigQuery and return JSON to the client

---

## Steps to Reproduce

1. Open the Incentive Compensation Tool from its Quick host (e.g. incentive-compensation-tool.quick.shopify.io).
2. Enter a valid CSM name.
3. Click **⚡ Load Live Data**.
4. **Observed:** Red error banner: `Failed to load data: Unexpected token '<', "<html> <h\"... is not valid JSON"`.
5. **Expected:** Either successful load of data or a clear, user-friendly error (e.g. “Data service unavailable” or “Permission denied”).

---

## Root Cause Analysis

### Client side (Incentive Comp Tool)

- The app loads **Quick.js** from `/client/quick.js` and uses **`quick.dw.querySync(sql)`** to run multiple BigQuery-style queries (see `index.html`):
  - One initial `querySync` for account lookup (e.g. ~line 1993).
  - Up to 9 further `querySync` calls in parallel for revenue, GMV, SP, GPV, etc. (e.g. ~lines 2177–2185).
- The Quick.js client (or the code path that handles `querySync` responses) **parses the response as JSON**. When the server returns a body that starts with HTML (e.g. `"<html> <h..."`), `JSON.parse` throws: `Unexpected token '<', "<html> <h\"... is not valid JSON"`.
- So the **immediate** cause is: **the response body is HTML where JSON is expected**.

### Server / backend side (data-portal-mcp or Quick data layer)

The actual HTTP response is not produced by this repo; it is produced by whatever serves the data warehouse API used by `/client/quick.js` (e.g. data-portal-mcp or the Quick platform’s backend). Likely causes include:

1. **Wrong route or 404:** The request hits a path that returns an HTML 404 page instead of the JSON query API.
2. **Server error (5xx):** An uncaught exception or proxy error returns an HTML error page instead of a JSON error payload.
3. **Auth/redirect:** A login or redirect page is returned as HTML when the client expects the JSON query result.
4. **Misrouting in production:** In some environments (e.g. local or non-Quick host), the request might hit a generic web server that always responds with HTML for unknown paths.

Because **https://github.com/Shopify/data-portal-mcp** returned **404** at report time, the following is based on typical API design; the actual implementation must be checked in the **data-portal-mcp** (or related) source.

---

## Relevant Source References (Incentive Comp Tool)

| Location | Code / behavior |
|----------|------------------|
| `index.html` ~L9 | `<script src="/client/quick.js"></script>` — loads Quick runtime |
| `index.html` ~L1918–1936 | `loadLiveData()` — checks for `quick` / `quick.dw`, then calls `quick.dw.querySync` |
| `index.html` ~L1993 | First `querySync`: account lookup |
| `index.html` ~L2177–2185 | Parallel `querySync` for revenue, GMV, SP, GPV, etc. |
| `index.html` ~L2556–2568 | `catch`: shows error; includes detection for “HTML instead of JSON” and a clearer message |

The **parse error** occurs inside the Quick.js / data-warehouse client when it tries to parse the response (e.g. `JSON.parse(responseBody)`). The exact line is in the Quick runtime or the backend client, not in `index.html`.

---

## Required changes (data-portal-mcp / Quick backend)

*These changes must be made in the service that serves the data warehouse API used by `quick.dw.querySync`. The Incentive Comp Tool cannot fix the HTML response on its own.*

1. **Ensure query API routes return only JSON**
   - All responses for the endpoint(s) used by `quick.dw.querySync` should have `Content-Type: application/json` and a JSON body.
   - No HTML error pages should be returned for these API routes.

2. **Use JSON error responses for 4xx/5xx**
   - For auth errors, “not found”, rate limits, or server errors, return a JSON body, e.g.:
     - `{"error": "unauthorized", "message": "..."}`
     - `{"error": "internal_error", "message": "..."}`
   - Avoid returning HTML for any path that the Quick client calls for query results.

3. **Routing and environment**
   - Verify that in the environment where the Quick app runs (e.g. incentive-compensation-tool.quick.shopify.io), the request from the browser reaches the **data-portal / BigQuery proxy** and not a generic HTML-serving route.
   - If the backend is behind a reverse proxy or API gateway, ensure that errors from the proxy are also returned as JSON for these endpoints.

4. **Logging**
   - On the server, log request URL, method, and response status when returning non-2xx or non-JSON (e.g. when the body starts with `<!` or `<html`) to make future debugging easier.

---

## Client-Side Mitigation (already in place)

The Incentive Comp Tool has been updated to:

- Check for the presence of `quick` and `quick.dw` before calling `querySync`, and show a clear message if the app is not running in the Quick environment.
- In the `loadLiveData()` catch block, detect “HTML instead of JSON” (e.g. via `is not valid JSON`, `Unexpected token.*<`, or `<html`) and show a short, actionable message directing users to open the app from the correct Quick/Shopify environment or to use “Load from JSON” for local/testing.

This does not fix the server returning HTML; it only improves the user experience when that happens.

---

## Checklist for data-portal-mcp (or owning team)

*Use this list when implementing the required changes above.*

- [ ] Confirm the HTTP endpoint(s) used by `quick.dw.querySync` and ensure they always respond with JSON (or a documented non-JSON behavior).
- [ ] Replace any HTML error pages for these endpoints with JSON error responses.
- [ ] Add or review routing so that the Quick app’s requests never hit an HTML-only error page.
- [ ] Add server-side logging when the response is HTML or non-JSON for an API path.
- [ ] (Optional) Expose a small health or “query capability” check that returns JSON, so clients can detect “wrong environment” or “data portal down” without parsing HTML.

---

## Appendix: Error message and parsing

- **User-visible:** `Failed to load data: Unexpected token '<', "<html> <h\"... is not valid JSON"`
- **Typical cause:** `JSON.parse(responseBody)` where `responseBody` starts with `"<html> <h"` (or similar HTML).
- **Conclusion:** The server (or an intermediary) returned an HTML document instead of the expected JSON for the data warehouse query API.
