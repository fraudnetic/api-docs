# Fraudnetic Partner Integration — V2 Collector Service (Fraud Engine)

**Fraudnetic** · Partner documentation · 2026-05-15

Copyright © 2025–2026 Fraudnetic, Inc. All rights reserved. See [LEGAL.md](../../LEGAL.md).

Machine-readable contract: [openapi.yaml](./openapi.yaml)

This guide covers the **Fraud Engine Collector** ingest API used to send events, customers, and transactions into Fraudnetic for scoring. Dashboard **Rules** management is internal and is not part of this partner surface.

---

## 1. Preface

Integrate with the Fraudnetic Collector API: work cycle, request signing, headers, payload fields, examples, and errors.

## 2. Dictionary

| Term | Meaning |
|------|---------|
| FN | Fraudnetic |
| P | Partner |
| PS | Partner Server |
| PF | Partner Frontend |
| CS | Collector Service |
| PUB | Fraudnetic project public key |
| PRIV | Fraudnetic project private key |
| XT | Frontend fingerprint data/token |
| Front-end Library | [fraudneticagent-v2](https://www.npmjs.com/package/fraudneticagent-v2) |

## 3. Work cycle

1. PF collects device/session/fingerprint context.
2. PS builds collector JSON payload.
3. PS signs payload with `Hash` + `Timestamp`.
4. PS sends request to the collector endpoint.

## 4. Environments and credentials

| Variable | Purpose |
|----------|---------|
| `{COLLECTOR_BASE_URL}` | Collector base URL |
| `PUB` / `PRIV` | Project signing keys |

Hash must be generated from the **exact compact JSON** sent to the endpoint.

## 5. Authentication and signing

### Required headers

| Header | Notes |
|--------|-------|
| `Timestamp` | Required. Recommended: Unix milliseconds string |
| `Hash` | Required. Base64 SHA-256 digest |

Send `Content-Type: application/json` with the request body. It is not part of the hash input and is not validated as an auth header.

### Hash formula

```text
Hash = base64(sha256(compact_json_payload + PRIV + PUB + Timestamp))
```

The service first tries a compacted JSON body, then falls back to the raw body bytes.

### Signing rules

- Hash the exact JSON body that is sent (prefer compact JSON)
- Use the same `Timestamp` value in the hash input and HTTP header
- Recommended timestamp format: Unix milliseconds string

```js
import crypto from "crypto";

export function buildCollectorHeaders(bodyObject, pub, priv) {
  const timestamp = Date.now().toString();
  const payload = JSON.stringify(bodyObject);
  const hash = crypto
    .createHash("sha256")
    .update(payload + priv + pub + timestamp)
    .digest("base64");
  return {
    headers: {
      "Content-Type": "application/json",
      Timestamp: timestamp,
      Hash: hash,
    },
    payload,
  };
}
```

## 6. Collector API

Base URL: `{COLLECTOR_BASE_URL}`

### Common response shapes

**Success**

```json
{ "data_processing": true, "error_message": "" }
```

Some accepted-but-not-queued event names may return `{ "data_processing": true }` without `error_message`.

**Business rejection (HTTP 200)**

```json
{ "data_processing": false, "error_message": "Duplicate event" }
```

Also used for duplicate customer / bonus / action payloads.

**Error**

```json
{ "error_message": "..." }
```

Missing required event fields typically return **HTTP 401**. Invalid hash returns **401**. Other validation failures return **400**.

### Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/v2/health` | Service health (`{ "status": <IPInfo> }`) |
| POST | `/v2/event` | Ingest behavioral / lifecycle events |
| POST | `/v2/customer` | Upsert customer profile |
| POST | `/v2/transaction` | Ingest monetary transactions |
| POST | `/v2/bonus` | Optional bonus definitions |
| POST | `/v2/actions` | Optional action definitions |

See [openapi.yaml](./openapi.yaml) for full schemas and examples.

#### Accepted `event_name` values

`login`, `registration`, `logout`, `accountUpdate`, `emailVerification`, `documentVerification`, `passwordReset`, `emailChange`, `phoneNumberChange`, `deposit`, `withdrawal`, `sportBet`, `casinoSpin`, `gameBet` (also `phoneNumberVerification` per service).

Some verification/password/email/phone change events may be accepted but not queued for scoring.

#### Transaction required fields

`customer_id`, `client_transaction_id`, `currency`, `transaction_type`, and `amount` (must be `> 0`).

#### Transaction types

`DEPOSIT`, `WITHDRAW`, `BONUS_IN`, `BONUS_OUT`, `BET`, `WIN`, `SPORT_BET`, `SPORT_WIN`, `P2P_GAME_BET`, `P2P_GAME_WIN`, `BUY`, `SELL` (case-insensitive).

`session` is accepted as an alias for `session_id` on event, customer, and transaction payloads.

## 7. Integration checklist

- [ ] Store PUB and PRIV securely
- [ ] Generate Timestamp and Hash per request
- [ ] Hash the exact compact JSON body
- [ ] Send all required payload fields
- [ ] Include traffic tagging (`affiliate_id`, `affiliate_name`, `btag`) when available

## 8. Troubleshooting — invalid hash

Common causes:

- Payload used for hash differs from payload sent
- Wrong key order in hash input (`body + PRIV + PUB + Timestamp`)
- Timestamp in header differs from signing timestamp
- Wrong project credentials

## 9. OpenAPI / Scalar

Import `openapi.yaml` into Scalar or open the repo [preview](../../preview/index.html).
