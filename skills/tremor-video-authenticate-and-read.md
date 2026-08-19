---
name: Authenticate against the Nexxen DSP API and read a resource list
description: >-
  Obtain a bearer token from the Nexxen Token Service, then list any Campaign Management resource
  correctly — handling the 300 Multiple Choices market-disambiguation response, the
  offset/limit/sortOrder/sortKey pagination contract, and the gateway rate limit.
api: collections/tremor-video.postman_collection.json
contract: Postman Collection v2 (Nexxen DSP APIs) — the provider publishes no OpenAPI
operations:
  - New Bearer Token
  - List of Advertisers
  - List of Account Managers
  - Existing Advertiser
generated: '2026-08-13'
method: generated
source: collections/tremor-video.postman_collection.json
---

# Authenticate against the Nexxen DSP API and read a resource list

Every Nexxen DSP API service (Campaign Management, Reporting, Device, Location) sits behind the
same Amobee Cloud Gateway at `https://services.amobee.com` and takes the same bearer token. The
Token Service is the only service that does not require one.

## 1. Get a bearer token

`New Bearer Token` — `POST https://services.amobee.com/accounts/v1/api/token`

Send the credentials in the JSON body (not as a Basic header, and not as form fields):

```json
{ "client_id": "<your service account id>", "client_secret": "<secret>", "grant_type": "client_credentials" }
```

Send every subsequent call with `Authorization: Bearer <access_token>`. A missing or expired token
returns `401` with
`{"data":[],"errors":[{"message":"The \"Authorization\" header ... is missing, or specifies an expired or invalid access token.","errorCode":1,"statusCode":"UNAUTHORIZED"}]}`.
Re-send this same POST to replace an expired token — the credentials do not change.

Calling the token endpoint with anything other than POST returns `405 Method Not Allowed`.

## 2. Resolve the market before you list anything

`List of Advertisers` — `GET {campaign_url}/advertisers?marketId=1&offset=0&limit=2&sortOrder=asc&sortKey=name`
where `campaign_url` = `https://services.amobee.com/campaign/v5/api`.

If the credential has access to more than one market and `marketId` is omitted, the API returns
**`300 Multiple Choices`** — not an error body, but a `links.markets` map of every market ID to a
ready-made URL:

```json
{"data":[],"errors":[],"links":{"markets":{"1":"https://services.amobee.com/campaign/v5/api/accountManagers?marketId=1","2":"...?marketId=2"}}}
```

Treat a 300 as "pick a market and re-send" — follow one of the `links.markets` URLs verbatim.
Resolve the market once and reuse it; this applies to `accountManagers`, `advertisers`,
`insertionOrders`, `packages`, `lineItems`, `segments`, `segmentSets`, `concepts`, `conceptSets`,
`beacons`, `inventoryLists`, `locationGroups`, `publisherDeals` and `creatives`.

## 3. Paginate and sort

List operations take `offset`, `limit`, `sortOrder` (`asc`/`desc`) and `sortKey` (a field of the
resource, e.g. `name`, `id`, `cost`). The response carries `queryTotal` — the total matching rows
— alongside `data[]`, so page with `offset += limit` until `offset >= queryTotal`. There are no
cursors and no `Link` header.

## 4. Do not combine scope filters

`marketId`, `advertiserId`, `insertionOrderId` and `packageId` are **mutually exclusive** on list
requests. Sending more than one is rejected. Choose the narrowest single scope you need.

## 5. Respect the rate limit

The gateway returns `X-RateLimit-Remaining`, `X-RateLimit-Burst-Capacity` and
`X-RateLimit-Replenish-Rate` twice on every response — a per-day bucket (86400) and a per-second
bucket (8/s). Exhaustion returns **`429` with an empty body and no `Retry-After`**, so back off on
the header, not on a retry hint. See `rate-limits/tremor-video-rate-limits.yml`.

## 6. No idempotency keys

The API documents no idempotency key. A retried `POST` creates a second object — re-read with the
matching `List of ...` request before retrying a create.
