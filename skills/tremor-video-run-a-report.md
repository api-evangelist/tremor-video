---
name: Run a Nexxen DSP performance report
description: >-
  Submit an asynchronous report to the Nexxen Reporting Service, read the report id off the
  Location header, and poll for the result inside its 24-hour availability window.
api: collections/tremor-video.postman_collection.json
contract: Postman Collection v2 (Nexxen DSP APIs) — the provider publishes no OpenAPI
operations:
  - New Report
  - Report Details
generated: '2026-08-13'
method: generated
source: collections/tremor-video.postman_collection.json
---

# Run a Nexxen DSP performance report

Base: `reporting_url` = `https://services.amobee.com/reporting/v2/api`. Same bearer token as the
Campaign Management service. The Reporting Service is scoped to quick, limited-scope metric pulls —
the collection is explicit that it does not replace large-scale reporting inside the DSP UI.

## 1. Submit the report

`New Report` — `POST {reporting_url}/reports?marketId=1`

The body describes the query, not the output rows:

```json
{
  "aggregationType": "summary",
  "startTime": "2026-06-22T00:00:00.000Z",
  "endTime": "2026-06-24T00:00:00.000Z",
  "outputPath": "",
  "filters": [ { "objectType": "advertiser", "...": "..." } ]
}
```

- Times are ISO-8601 UTC.
- `filters[]` scopes the pull by `objectType` (advertiser, insertion order, line item, …).
- `outputPath` may target an S3 destination; leave empty to retrieve inline.
- Each metric may carry an `outputName` to rename the column; it defaults to the metric `name`.

The call is **asynchronous** and answers `202 Accepted` with an empty body. The generated report id
is appended to the URL in the **`Location` response header** — parse it from there, not from the
body.

## 2. Poll for the result

`Report Details` — `GET {reporting_url}/reports/{id}`

Poll until the report is ready. The collection notes each report request is asynchronous with no
concurrency constraint, so several may be in flight at once — but the 8 requests/second gateway
bucket still applies to the polling itself.

## 3. Respect the documented limits

| Limit | Value |
|---|---|
| Calls | 10,000 per day, 8 per second |
| Data lookback | 19 months maximum |
| Output file size | 300 MB maximum for `outputPath` to S3 |
| Result retention | 24 hours |

Past the retention window the poll returns `404`:

```json
{"data":[],"errors":[{"messages":[{"type":"other","code":"1","message":"Requested report is no longer available, 24 hour availability window has passed."}]}]}
```

That is not a transient failure — re-submit `New Report` and poll the **new** id. A different `404`
shape, `{"errors":[{"message":"error_message_not_found"}]}`, means the id itself is unknown.

Exhausting the rate limit returns `429` with an empty body and no `Retry-After`; read
`X-RateLimit-Remaining` off the previous response instead. See
`rate-limits/tremor-video-rate-limits.yml`.
