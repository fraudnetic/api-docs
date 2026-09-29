# Fraudnetic API Docs (public preview)

Live Scalar reference for partner-facing **KYC** and **Fraud Engine Collector** APIs.
This repository is the **source of truth** for what partners are told about these
two APIs; everything else copies from here.

**Copyright © 2025–2026 Fraudnetic, Inc.** — see [LEGAL.md](./LEGAL.md).

## Preview

| Page | URL |
|------|-----|
| Scalar UI | https://fraudnetic.github.io/api-docs/preview/ |
| Site root | https://fraudnetic.github.io/api-docs/ |
| KYC OpenAPI | https://fraudnetic.github.io/api-docs/products/kyc/openapi.yaml |
| Collector OpenAPI | https://fraudnetic.github.io/api-docs/products/fraud-engine/openapi.yaml |
| Bundled JSON | `…/bundled/kyc.openapi.json`, `…/bundled/fraud-engine.openapi.json` |

Source of truth for internal Rules (private): https://github.com/fraudnetic/documentation

## Layout

```
products/kyc/openapi.yaml            KYC partner API (OpenAPI 3.1)
products/kyc/guide.md                Narrative integration guide
products/fraud-engine/openapi.yaml   Collector ingest API (OpenAPI 3.1)
products/fraud-engine/guide.md       Narrative integration guide
preview/index.html                   Scalar page published to GitHub Pages
redocly.yaml                         Lint rules
```

## Working on the specs

```bash
npm install
npm run lint      # validate both documents (same check CI runs)
npm run bundle    # resolve $refs into dist/*.openapi.json
npm run preview   # serve the repo at http://localhost:4000/preview/
```

`npm run lint` is a required gate: the Pages workflow refuses to publish a
document that does not validate.

## Who consumes these documents

| Consumer | How it reads them | Drift check |
|---|---|---|
| GitHub Pages preview | directly from `products/` | — |
| Dashboard front (`kyc-v1-fraud-v3-front`) | copied into `public/openapi/` by `yarn sync:api-specs` | `tests/unit/api-docs/openapiSpecs.test.ts` |
| KYC backend (`v1-kyc-backend`) | read at test time from `../api-docs` | `routes/openapi_contract_test.go` |
| Collector (`v2-collector-microservice`) | read at test time from `../api-docs` | `internal/apicontract/openapi_contract_test.go` |

The two Go test suites are real contract tests: they replay the published request
examples against the actual handlers and validate the handlers' responses against
the published schemas. Check both repositories out next to this one, or point
`KYC_OPENAPI_SPEC` / `COLLECTOR_OPENAPI_SPEC` at the documents. They skip, rather
than fail, when this repository is not available.

## Changing an API

1. Change the handler.
2. Change the document here and run `npm run lint`.
3. Run the service's contract test — it fails if the document and the handler disagree.
4. Run `yarn sync:api-specs` in the dashboard front so the embedded reference matches.

## Scope

Only partner-facing APIs are documented here: **KYC** and **Fraud Engine Collector**.
Internal services (score engine, control, dashboard backend, automation, exporter)
are deliberately out of scope, as are the Collector's operational endpoints
(`/v2/metrics`, `/v2/metrics/rps`) and the KYC admin/dashboard endpoints under
`/v1/admin/…`.
