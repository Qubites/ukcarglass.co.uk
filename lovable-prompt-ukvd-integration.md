# Lovable Prompt — UK Vehicle Data (VehicleDetailsWithImage) Integration

You are building the new UK windscreen booking website.

## Goal
Implement a **secure server-side** integration to UK Vehicle Data Global (`https://uk.api.vehicledataglobal.com/r2/lookup`) to resolve a user-entered **UK registration number (VRM)** (or VIN/UVC/UKVDID) into a normalized **Vehicle Profile** used by the booking flow.

## Non-negotiables
1. **Do NOT expose the UKVD ApiKey in the browser.** All calls must go through a server route (API route / edge function).
2. Add **basic validation + normalization** for VRM (uppercase, trim, remove extra spaces).
3. Add **caching** (24h by VRM) to reduce cost and rate limits.
4. Handle **429** with a friendly “try again later” message and backoff.
5. Do not block the booking journey on image download. Images are optional.

## Environment variables
Create these env vars (server-side only):
- `UKVD_API_BASE_URL=https://uk.api.vehicledataglobal.com`
- `UKVD_API_KEY=<secret uuid>`
- `UKVD_PACKAGE_NAME=VehicleDetailsWithImage`

## Backend API you must create
Create a server endpoint:

### `GET /api/vehicle/lookup?vrm=AA12AAA`
- Accepts exactly one of: `vrm`, `vin`, `uvc`, `ukvdId`.
- Calls UKVD `/r2/lookup` with query params:
  - `ApiKey` = `UKVD_API_KEY`
  - `PackageName` = `UKVD_PACKAGE_NAME`
  - plus the selected search param (`Vrm` / `Vin` / `Uvc` / `UkvdId`)
- Returns a **normalized** response object for the frontend.

### Normalized response shape (what frontend should rely on)
Return JSON:

```json
{
  "ok": true,
  "source": "ukvd",
  "search": { "type": "vrm", "value": "AA12AAA" },
  "vehicle": {
    "vrm": "AA12AAA",
    "vin": "…",
    "make": "…",
    "model": "…",
    "year": 2020,
    "bodyStyle": "…",
    "doors": 5
  },
  "images": [
    { "viewAngle": "Front", "url": "https://…", "expiresAt": "2025-01-01T00:00:00Z" }
  ],
  "meta": {
    "statusCode": 0,
    "statusMessage": "Success",
    "responseId": "…",
    "queryTimeMs": 123
  },
  "raw": null
}
```

If not successful, return:

```json
{
  "ok": false,
  "error": {
    "code": "NO_MATCH" | "RATE_LIMIT" | "UNAUTHORIZED" | "BAD_INPUT" | "UPSTREAM_ERROR",
    "message": "Human friendly message"
  },
  "meta": { "statusCode": 26, "statusMessage": "NoMatchFound", "responseId": "…" }
}
```

## Status mapping rules (important)
Use `ResponseInformation.StatusCode` and map to these frontend error codes:
- `0` or `1` => ok=true
- `2`, `26` => `NO_MATCH`
- `7`, `22` or HTTP 401 => `UNAUTHORIZED`
- HTTP 429 or status `20`, `28` => `RATE_LIMIT`
- `13`, `14` => `BAD_INPUT`
- Everything else => `UPSTREAM_ERROR`

## VRM validation rules (simple + tolerant)
- Normalize: `vrm = vrm.trim().toUpperCase().replace(/\s+/g, '')`
- If after normalization length < 5 or > 8 => BAD_INPUT
- Do not hard-fail if regex doesn’t match; still allow API to validate (but show inline hint).

## Caching
- Cache by key: `vehicle_lookup:vrm:<VRM>` (or `vin:<VIN>`, etc.)
- TTL: 24 hours
- Cache the normalized response for success and also cache `NO_MATCH` for 1 hour (avoid repeated paid calls).

## Frontend integration
1. Create a small client helper: `lookupVehicle(search: { vrm?: string; vin?: string; uvc?: string; ukvdId?: string })`
2. In the booking flow:
   - Step A: user selects glass position (Front / Rear / Side) *(side needs extra confirmation later)*
   - Step B: user enters VRM (or VIN).
   - Step C: call `/api/vehicle/lookup`, show make/model/year confirmation card.
   - Step D: proceed to windscreen product mapping logic (VehicleDataGlobal -> AutoglassCRM -> WhatWindscreen) without exposing keys.
3. Persist the normalized `vehicle` object in session (or local storage) to avoid re-lookups.

## Deliverables you must implement
- Server route `/api/vehicle/lookup`
- Strong typing (TypeScript interfaces) for normalized response
- Unit-ish guard checks for inputs
- Clear error handling + UI states (loading, success card, error card)
- No API key leakage anywhere (no client logs, no network calls direct)

## Testing checklist
- Valid VRM returns make/model/year
- Unknown VRM returns NO_MATCH with friendly message
- Rate limit returns RATE_LIMIT with “try again in a moment”
- API key missing => UNAUTHORIZED and logged server-side only
- Caching reduces repeated calls (verify by logging cache hits server-side)
