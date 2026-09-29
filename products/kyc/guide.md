# Fraudnetic Partner Integration — V1 KYC Service

**Fraudnetic** · Partner documentation · 2026-05-15

Copyright © 2025–2026 Fraudnetic, Inc. All rights reserved. See [LEGAL.md](../../LEGAL.md).

Machine-readable contract: [openapi.yaml](./openapi.yaml)

---

## 1. Preface

This guide helps developers integrate with the Fraudnetic KYC API. It covers the work cycle, partner authentication, request fields, examples, and error handling.

## 2. Dictionary

| Term | Meaning |
|------|---------|
| FN | Fraudnetic |
| P | Partner |
| PS | Partner Server (your backend) |
| PF | Partner Frontend |
| KS | KYC Service |

## 3. Work cycle

1. PS calls `POST /v1/users` with `X-Partner-API-Key`.
2. KS returns `session.token`, `verificationId`, and `redirectUrl`.
3. PS redirects (or embeds) the user using `redirectUrl` (`step=start`, `session_token`, `verificationId`, optional `iframe` / `lang`).
4. User completes document and face capture in the KYC UI (session-authenticated uploads).
5. Partner receives the result via webhook push and/or pulls:
   - `GET /v1/partner/verifications/{verificationId}/latest`
   - `GET /v1/partner/verifications/{verificationId}/webhook/latest`

## 4. Environments and credentials

| Variable | Purpose |
|----------|---------|
| `{KYC_BASE_URL}` | KYC API base URL (include host; paths below start with `/v1`) |
| `{KYC_PARTNER_API_KEY}` | Partner API key for `X-Partner-API-Key` and HMAC signing |

## 5. Authentication

### 5.1 Create session

Send `X-Partner-API-Key: {KYC_PARTNER_API_KEY}` on `POST /v1/users`.

### 5.2 Session uploads / status

Send `Authorization: <session.token>` as the raw session token value returned by `POST /v1/users` (HMAC-signed token; do not prefix with `Bearer` unless your deployment explicitly supports it).

### 5.3 Partner verification pull (HMAC)

Server validation uses **only** timestamp + signature. The HMAC key is the partner API key.

| Header | Required | Notes |
|--------|----------|-------|
| `X-Fraudnetic-Timestamp` | Yes | Unix seconds; max skew 60s |
| `X-Fraudnetic-Signature` | Yes | `sha256=` + hex HMAC-SHA256 |
| `X-Partner-API-Key` | No | Not read by the pull middleware; optional for your own tracing |

**Signature message:** `{timestamp}.{METHOD}.{path}`  
**Key:** partner API key  
**Example path:** `/v1/partner/verifications/{verificationId}/latest`

```js
import crypto from "crypto";

export function buildPartnerPullHeaders(apiKey, method, path) {
  const ts = Math.floor(Date.now() / 1000).toString();
  const message = `${ts}.${method}.${path}`;
  const sig = crypto.createHmac("sha256", apiKey).update(message).digest("hex");
  return {
    "X-Fraudnetic-Timestamp": ts,
    "X-Fraudnetic-Signature": `sha256=${sig}`,
  };
}
```

### 5.4 Partner webhook (push)

When a partner webhook endpoint is configured in the KYC admin (target URL + **endpoint secret**), KS posts the final verification status with:

- `X-Fraudnetic-Event-Type` (e.g. `kyc.verification.status_changed`)
- `X-Fraudnetic-Event-Id`
- `X-Fraudnetic-Timestamp` (RFC3339 on production deliveries)
- `X-Fraudnetic-Signature` = `sha256=` + hex HMAC of **raw request body**

**Signing secret:** the per-endpoint webhook `secret` stored with the webhook configuration — this may differ from `{KYC_PARTNER_API_KEY}`.

**Retries:** on non-2xx responses, KS retries with exponential backoff (approximately 1s, 2s, 4s, … up to a 5-minute cap; default max retries is 6).

## 6. KYC API

See [openapi.yaml](./openapi.yaml) for full schemas. Summary:

### `POST /v1/users`

Creates or resumes a KYC session.

**Required:** `returnUrl`, `user_data.userID`  
**Optional:** `language`, `iframe`, `verificationId`, `workflowId`, and additional `user_data` fields (`email`, `phone`, `sex`, `name`, `surname`, `dateOfBirth`, …)

**Success** includes `verificationId`, `status` (`started`), `redirectUrl`, and `session.token`.

### Session uploads

- `POST /v1/kyc/uploads/documents` — document images (base64 or data URL). Provide `country`, `documentType`, and at least one of `frontDocument` / `backDocument` (passport may be back-only). `verificationId` may be omitted when it is bound to the session token. Supported types include `national_id`, `passport`, `driving_licence`, `residence_card`, `health_card`.
- `POST /v1/kyc/uploads/face` — live face capture (JSON or multipart)

### Partner pull

- `GET /v1/partner/verifications/{verificationId}/latest` — latest raw verification data
- `GET /v1/partner/verifications/{verificationId}/webhook/latest` — latest webhook delivery / payload

### Status (session)

- `GET /v1/kyc/verifications/{verificationId}/status` — poll the current status and human-readable rejection reasons
- `GET /v1/kyc/{verificationId}/check-status` — returns `finalized` once the verification reaches a terminal status

Both endpoints return `403` when the session token is bound to a different
verification than the one in the path.

### Timeline events (session)

- `GET /v1/kyc/verifications/{verificationId}/events` — the verification timeline
- `POST /v1/kyc/verifications/{verificationId}/events` — append an event (the hosted UI reports `opened` this way)

These are used by the hosted verification UI. Partners normally do not need them.

### Service

- `GET /v1/health` — unauthenticated liveness probe, returns `{"status":"ok"}`

### Legacy combined upload

`POST /v1/kyc/uploads` accepts documents and the face capture in one request using
**snake_case** field names (`document_type`, `front_document`, `back_document`,
`live_face`). It stores the files without running the decision engine inline.
New integrations should use `/v1/kyc/uploads/documents` and `/v1/kyc/uploads/face`.

## 6.1 Webhook payload

Event type `kyc.verification.status_changed`:

```json
{
  "event_type": "kyc.verification.status_changed",
  "event_id": "4f0a2f6a-4f0b-4f4a-9d4b-9d5a1c2f3e44",
  "created_at": "2026-09-29T10:15:30Z",
  "verification": {
    "id": "6b1f0f6e-2a0e-4f8f-9f0d-2f7b6b9a1c33",
    "status": "approved",
    "completed_at": "2026-09-29T10:15:29Z",
    "document_type": "national_id",
    "country": "AM"
  },
  "user": {
    "id": "2a1d9e8c-5b3f-4a7e-8c1d-0f9b2a3c4d5e",
    "partner_user_id": "abc-1001",
    "email": "user@example.com"
  },
  "personal_data": { "full_name": "John Doe", "date_of_birth": "1992-06-15" },
  "user_files": { "front_document": "https://…", "live_face": "https://…" },
  "rejection_reasons": [],
  "steps": [
    { "step": "front_document", "status": "approved", "rejection_reasons": [] },
    { "step": "live_face", "status": "approved", "rejection_reasons": [] }
  ]
}
```

Verifying the signature (Node.js):

```js
import crypto from "crypto";

export function verifyWebhook(rawBody, signatureHeader, endpointSecret) {
  const expected =
    "sha256=" +
    crypto.createHmac("sha256", endpointSecret).update(rawBody).digest("hex");
  return crypto.timingSafeEqual(
    Buffer.from(signatureHeader),
    Buffer.from(expected),
  );
}
```

`rawBody` must be the **unparsed** request body. Deduplicate on
`X-Fraudnetic-Event-Id`, because retries reuse the same id.

## 7. Integration checklist

- [ ] Call `/v1/users` before opening the KYC page
- [ ] Store `session.token` and `verificationId`
- [ ] Redirect or embed using `redirectUrl`
- [ ] Configure webhook target URL + endpoint secret and/or implement signed pull endpoints
- [ ] Validate inbound webhook HMAC using the **endpoint secret** (not necessarily the partner API key)
- [ ] Deduplicate webhook deliveries on `X-Fraudnetic-Event-Id`
- [ ] Reply `2xx` to webhooks quickly and process asynchronously

## 8. OpenAPI / Scalar

Import `openapi.yaml` into Scalar or open the repo [preview](../../preview/index.html).
