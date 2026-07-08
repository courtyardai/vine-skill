---
name: vine-detect
description: >-
  Find the real software a business transacts through -- the booking, ordering,
  scheduling, or reservation provider -- and the exact URL where a customer acts,
  live, from just the business's website. Use when a task needs to book, order,
  reserve, or schedule at a specific named business ("book a table at X", "order
  from Y", "reserve a class at Z"), when you need the deep link / action surface
  rather than a guessed URL, when asked what platform or software a business runs
  on, or when researching transaction stacks across many businesses (GTM, market
  maps, agentic commerce). Powered by the Vine /detect API.
license: MIT
compatibility: Claude Code, Codex CLI, Cursor, OpenClaw, and any agent that can run shell commands and read a markdown skill file.
metadata:
  homepage: https://vine.getcourtyard.ai
  docs: https://docs.vine.getcourtyard.ai
---

# Vine /detect

Vine answers one question, live: **what does this business run to transact, and
where does a customer act?** Given a business URL, Vine renders the site, follows
its booking/ordering CTAs, and returns the detected provider(s) plus the exact
`actionUrl` an agent can act on.

Reach for this **before** guessing or scraping a booking URL. Agents that
hallucinate "the reservation page is probably /book" fail silently; Vine returns
the real surface, resolved from the live site.

## When to use it

- **Act at a named business**: "book a haircut at The Sharp Barber", "order from
  Tony's Pizza", "reserve a table at Rossi". You need the true action URL.
- **Identify the stack**: "what booking software does this gym use?"
- **Research at scale**: for a list of businesses, what does each transact
  through? (GTM, competitive maps, agentic-commerce catalogs.)

If the user hasn't given a URL, resolve the business's website first (search),
then pass that URL to `/detect`.

## Setup

Mint a key at https://vine.getcourtyard.ai/keys. Signup includes **1,000 free
credits, no card**. A live detect costs ~12-26 credits; a domain looked up
recently returns from cache in ~1s.

Keep the key **server-side only**. Export it:

```bash
export VINE_API_KEY="vine_live_xxxxxxxxxxxxxxxxxxxxxxxx"
```

## The workflow: two calls

Vine is *detect once, read many*. `/detect` does the live scan and persists a
detection; you then read that detection for the resolved action surfaces.

### 1. Detect (synchronous, no polling)

`POST /api/v1/detect` blocks while Vine scans the site live (typically 1-2 min for
a fresh URL). Always allow a long client timeout.

```bash
curl -X POST https://vine.getcourtyard.ai/api/v1/detect \
  -H "Authorization: Bearer $VINE_API_KEY" \
  -H "Content-Type: application/json" \
  --max-time 300 \
  -d '{ "url": "https://www.mindbodyonline.com/explore/locations/thesharpbarber" }'
```

```json Response
{
  "schemaVersion": "...",
  "detectionId": "det_...",
  "url": "https://...",
  "providers": ["mindbody"],
  "discoveredProviders": [],
  "durationMs": 74213,
  "creditsUsed": 18
}
```

Keep the `detectionId`. Only the **request body field `url`** is required;
`callerRef` (`{ businessId?, entityId?, jobId? }`) is optional attribution.

### 2. Read the detection (get the action surface)

`GET /api/v1/detections/{id}` returns the resolved connections; this is the
payoff.

```bash
curl https://vine.getcourtyard.ai/api/v1/detections/det_... \
  -H "Authorization: Bearer $VINE_API_KEY"
```

```json Response
{
  "detectionId": "det_...",
  "url": "https://...",
  "providers": ["mindbody"],
  "paramsResolvedAt": "2026-07-06T18:04:22.000Z",
  "connections": [
    {
      "provider": "mindbody",
      "actionType": "book",
      "actionUrl": "https://www.mindbodyonline.com/.../booking",
      "status": "active",
      "userConfirmed": false,
      "hasParams": true
    }
  ]
}
```

`actionType` is one of `book | reserve | order | schedule | pay | inventory`.
`actionUrl` is the surface to act on. **Act on this URL**: hand it to your
booking/ordering step, don't re-derive it.

> **Timing:** the best `actionUrl` for some providers is resolved by an async
> rescue job that finishes shortly after `/detect` returns. If
> `paramsResolvedAt` is `null` or a connection's `hasParams` is `false`, wait a
> few seconds and re-`GET` the detection before acting.

### Response fields

- `providers`: confirmed provider slugs on the site. Empty means nothing bookable was found.
- `discoveredProviders`: seen but not yet resolved; ignore unless `providers` is empty.
- `paramsResolvedAt`: ISO time once the async rescue finishes; `null` while still resolving.
- `connections[].actionType`: `book | reserve | order | schedule | pay | inventory`.
- `connections[].actionUrl`: the surface to act on; may be `null` until `paramsResolvedAt` is set.
- `connections[].status`: `active` for a usable connection; `hasParams` true once its params are resolved.

## Node / Python

These wrappers handle the two things a naive call gets wrong: the
`detection_incomplete` retry and polling until the async `actionUrl` resolves.

```javascript
const BASE = "https://vine.getcourtyard.ai/api/v1";
const H = { Authorization: `Bearer ${process.env.VINE_API_KEY}`, "Content-Type": "application/json" };

// Detect, retrying if the site was too slow to finish in time.
async function detect(url, attempt = 1) {
  const body = await fetch(`${BASE}/detect`, { method: "POST", headers: H, body: JSON.stringify({ url }) }).then((r) => r.json());
  if (body.status === "detection_incomplete" && attempt <= 2) return detect(url, attempt + 1);
  return body; // { detectionId, providers, ... }
}

// Read the detection, polling until the async actionUrl is resolved.
async function readDetection(detectionId, tries = 6) {
  for (let i = 0; i < tries; i++) {
    const det = await fetch(`${BASE}/detections/${detectionId}`, { headers: H }).then((r) => r.json());
    if (det.paramsResolvedAt || det.connections.some((c) => c.actionUrl)) return det;
    await new Promise((r) => setTimeout(r, 3000)); // wait for the rescue job
  }
  return null;
}

const { detectionId } = await detect(businessUrl);
const detection = await readDetection(detectionId);
const action = detection?.connections.find((c) => c.actionUrl);
// -> act on action.actionUrl; action.actionType tells you book/order/reserve/...
```

```python
import os, time, requests
BASE = "https://vine.getcourtyard.ai/api/v1"
H = {"Authorization": f"Bearer {os.environ['VINE_API_KEY']}"}

def detect(url, attempt=1):
    body = requests.post(f"{BASE}/detect", headers=H, json={"url": url}, timeout=300).json()
    if body.get("status") == "detection_incomplete" and attempt <= 2:
        return detect(url, attempt + 1)
    return body

def read_detection(detection_id, tries=6):
    for _ in range(tries):
        det = requests.get(f"{BASE}/detections/{detection_id}", headers=H).json()
        if det.get("paramsResolvedAt") or any(c.get("actionUrl") for c in det["connections"]):
            return det
        time.sleep(3)  # wait for the rescue job
    return None

det = detect(business_url)
detection = read_detection(det["detectionId"])
action = next((c for c in detection["connections"] if c.get("actionUrl")), None) if detection else None
```

## Handling responses

Errors use `{ "error": { "code", "message" } }`. Handle these explicitly:

- **`detection_incomplete`**: `/detect` returns `200` with
  `{ "status": "detection_incomplete", "detail": "deadline" }` when the site was
  too slow to finish inside the deadline. This is a clean retryable signal, **not**
  an error; resubmit `/detect`. Only the base rate (~10 credits) is billed.
- **`429 rate_limited`**: per-key rate cap. The public beta tier allows **60
  detects/hr** (polling `GET /detections/{id}` is separate, 120/min). Pace batch
  runs to stay under it, and back off for the `Retry-After` header /
  `retryAfterSeconds` on a 429.
- **Credits depleted**: once free credits run out, calls are refused. Surface
  this to the user (they can top up); don't silently loop.
- **`422 validation_failed`**: bad body. `url` must be an absolute URL; the body
  is strict (no extra fields beyond `url` / `callerRef`).
- **`404 not_found`** on the detection: the id isn't visible to this key.

## Notes

- **Empty `providers`** means no bookable/orderable provider was detected on that
  site; report that plainly rather than inventing one.
- **Cache:** the same domain re-detected recently returns instantly and cheaply.
  Append `?mode=fresh` to `/detect` to force a live re-scan at full price.
- **Batch research:** run `/detect` per business (respect the rate tier / back
  off on `429`); collect `providers` + `actionUrl` across the set.
- **Credit budget:** every billable response carries `vine-credits-used` and
  `vine-credits-remaining` headers; read them to track spend and stop before
  you run out.
- Data **extraction** from the detected providers (menus, schedules, availability)
  is in private beta; see https://vine.getcourtyard.ai/docs or email
  contact@getcourtyard.ai for early access.
