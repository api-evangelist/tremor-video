---
name: Build a Nexxen DSP campaign end to end
description: >-
  Create the full Nexxen DSP campaign hierarchy — advertiser, insertion order, package, line item,
  creative and ad — in the order the API requires, then activate it with the status endpoints.
api: collections/tremor-video.postman_collection.json
contract: Postman Collection v2 (Nexxen DSP APIs) — the provider publishes no OpenAPI
operations:
  - New Advertiser
  - New IO
  - New Package
  - New LI (2 Level Campaign)
  - New LI (3 Level Campaign)
  - Upload Display Asset
  - New Creative
  - New Ad
  - Update LI Status
  - Update IO Status
generated: '2026-08-13'
method: generated
source: collections/tremor-video.postman_collection.json
---

# Build a Nexxen DSP campaign end to end

Base: `campaign_url` = `https://services.amobee.com/campaign/v5/api`. Authenticate first — see
`tremor-video-authenticate-and-read.md`. Every create is a `POST` **nested under its parent**, so
the parent id must exist before the child call.

## The hierarchy

```
advertiser → insertionOrder → (package) → lineItem → ad → creative
```

Nexxen supports a **2-level** campaign (line items directly under the IO) and a **3-level**
campaign (line items under a package under the IO). Pick one before you start — the create call is
different.

## 1. Advertiser

`New Advertiser` — `POST {campaign_url}/accounts/{accountId}/advertisers`

The advertiser is created under an **account**, not under a market. Read the account id from your
platform provisioning. `List of Advertisers` confirms it, and `Existing Advertiser`
(`GET {campaign_url}/advertisers/{advertiserId}`) returns the full object, including
`viewabilityProviders` and `identityPlatformVendor`.

## 2. Insertion order

`New IO` — `POST {campaign_url}/advertisers/{advertiserId}/insertionOrders`

`Copy Existing IO` — `POST {campaign_url}/insertionOrders/{id}?copy=self` — clones a known-good IO
rather than rebuilding its settings; prefer it when replicating a flight.

## 3. Package (3-level campaigns only)

`New Package` — `POST {campaign_url}/insertionOrders/{insertionOrderId}/packages`
`Copy Existing Package` — `POST {campaign_url}/packages/{id}?copy=self`

## 4. Line item

- 2-level: `New LI (2 Level Campaign)` — `POST {campaign_url}/insertionOrders/{insertionOrderId}/lineItems`
- 3-level: `New LI (3 Level Campaign)` — `POST {campaign_url}/packages/{packageId}/lineItems`
- Clone: `Copy Existing LI` — `POST {campaign_url}/lineItems/{id}?copy=self`

Targeting attaches here. Resolve the ids you need first from the read-only reference services:

- audiences: `List of Segments` / `List of Segment Sets`
- retargeting: `List of Retargeting Segments`
- inventory: `List of Inventory Sources`, `List of Inventory Lists`, `List of Deals`
- geo: `List of Location Groups`, plus the Location service (`countries`, `regions`, `cities`, `dmas`, `places`)
- device: the Device service (`deviceTypes`, `operatingSystems`, `manufacturers`, `devices`)

Do **not** use `List of Concepts` / `List of Concept Sets` for new work — the collection states
concepts are deprecated from the Nexxen DSP and should not be used, even though they still resolve.

## 5. Creative and ad

1. Upload the asset: `Upload Display Asset` / `Upload Video Asset` / `Upload Audio Asset` —
   `POST {campaign_url}/creativeAssets/{display|video|audio}?marketId=1`
2. Wrap it in a creative: `New Creative` —
   `POST {campaign_url}/advertisers/{advertiserId}/creatives?validateClickUrl=true&requirePreviewApproval=true`
3. Attach creatives to the line item: `New Ad` — `POST {campaign_url}/lineItems/{lineItemId}/ads`,
   body carries `creatives[]` with a `creativeId` + `weight` per creative and a `rotation` mode
   (e.g. `weighted`):

```json
{ "name": "2022 summer ad", "creatives": [{"creativeId": "...", "weight": 0.75}], "rotation": "weighted" }
```

Use `List of Ad Sizes` (`GET {campaign_url}/adSizes?mediaChannel=audio`) to pick a valid size.

## 6. Activate — status is a separate call

Creation does not start delivery. Each object has a dedicated status endpoint taking a small body:

| Object | Request | Body |
|---|---|---|
| Advertiser | `PUT {campaign_url}/advertisers/{advertiserId}/statuses` | `{"status":"pause"}` |
| IO | `PUT {campaign_url}/insertionOrders/{id}/statuses` | `{"status":"pause"}` |
| Package | `PUT {campaign_url}/packages/{id}/statuses` | `{"status":"pause"}` |
| Line item | `PUT {campaign_url}/lineItems/{id}/statuses` | `{"status":"pause"}` |
| Creative | `PUT {campaign_url}/creatives/{id}/statuses` | `{"status":"stop"}` |
| Ad | `PUT {campaign_url}/ads/{adId}/statuses` | `{"clientStatus":"pause"}` |
| Beacon | `PUT {campaign_url}/beacons/{id}/statuses` | `{"workflowStatus":"archived"}` |

Note the field name differs by object — `status`, `clientStatus`, `workflowStatus`. Read the object
back (`Existing ...`) to confirm; `status: "play"` is the running state seen on live objects.

## 7. Errors

`Edit ...` returns `204 No Content` on success. Reads return the standard envelope
(`queryTotal` / `data` / `errors` / `links`). A `300` on a list means "specify `marketId`". See
`errors/tremor-video-problem-types.yml`. No idempotency key exists, so never blind-retry a `POST`.
