---
name: Check CADC supply and reserve backing
description: >-
  Verify the live circulating supply of the CADC Canadian-dollar stablecoin across
  chains and confirm it is backed by independently attested reserves, using Loon's
  public Transparency API. No authentication required.
api: openapi/loon-finance-transparency-openapi.yml
operations:
  - getSupply
  - listAttestations
  - listIssuances
generated: '2026-07-20'
method: generated
---

# Check CADC supply and reserve backing

CADC is a fiat-backed stablecoin pegged 1:1 to the Canadian dollar, issued by Loon.
The Transparency API is public and unauthenticated (base URL `https://loon.finance`,
JSON, CORS `*`). Use it to answer "how much CADC exists and is it backed?"

## Conventions
- All calls are safe HTTP `GET`s — no auth header, no idempotency key.
- Amounts are returned as decimal **strings**; parse with a big-decimal type, never a float.
- Responses are edge-cached ~5 minutes.

## Steps

1. **Get current supply** — call `getSupply` (`GET /api/supply`).
   - Read `totals.circulatingCadc` for the network-wide circulating amount and
     `totals.totalSupplyCadc` for total minted-minus-burned.
   - Iterate `chains[]` for the per-chain breakdown (`chainName`, `cadcAddress`,
     `circulatingCadc`, `frozenCadc`). Note `chainId` is `null` for Solana.
   - `cadUsdPrice` / `cadUsdPriceSource` give the CAD→USD rate used for nominal USD values.

2. **Confirm reserve backing** — call `listAttestations` (`GET /api/attestations`).
   - Take the newest entry (`uploadedAt`); read `amount` (attested CAD reserves) and
     `auditor` (e.g. HDCPA Professional Corporation).
   - Compare the attested `amount` against `totals.circulatingNominalCad` from step 1
     to sanity-check 1:1 backing. `url` is a time-limited signed PDF link.

3. **(Optional) Inspect recent mints for one chain** — call `listIssuances`
   (`GET /api/issuances?chainId={id}&limit={n}`).
   - `chainId` is **required** (400 with a `{message}` error if omitted).
   - Each `issuances[]` item has `txHash`, `issuedAt`, `to`, and `amountCadc`.

## Errors
Failures return `{ "message": "<reason>" }` with an HTTP 4xx status (e.g. 400 when
`chainId` is missing on `listIssuances`). There is no RFC 9457 problem envelope.
