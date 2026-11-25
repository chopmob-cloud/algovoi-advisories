# 2026-11-25: unscreened sanctions verdict and signed compliance receipt

Status: fixed and deployed to production. Public documentation corrected the same day.
Found by: kta1kri, https://github.com/kta1kri
Reported: 2026-11-25, 15:46 UTC

## Summary

`POST /compliance/screen` on api.algovoi.co.uk returned `verdict: "allow"` with `sanctions_clear: true` and signed a compliance receipt with `screen_result: "ALLOW"` for the wallet `0x098B716B8Aaf21512996dC57EB0615e2383E2f96` on `eip155:8453`.

That address is on OFAC's SDN list as published 23 November 2026, under LAZARUS GROUP, program DPRK3, recorded as its ETH digital currency address. Sanctions screening was switched off on the deployment at the time, so no list was read.

The receipt was cryptographically valid. It verified offline against the published `did:key`, and its `payer_ref` was the SHA-256 of the network and address, so the document was bound to that wallet and also false.

## How it was found

Found while reviewing the public API with unauthenticated requests. The false clearance needed one request with a public SDN address, and no payment was made.

## Cause

`screen_wallet()` returned `hit=False` when the screening switch was off, and returned the same value when the address was empty or the database query failed. The result carried nothing to say whether a sanctions list had been consulted, so `/compliance/screen` read the absence of a match as a clearance, aggregated it to `allow`, and signed it. The same conflation set `sanctions_clear: true` on the unsupported network path.

## Impact

A caller that gated a payout on the verdict or on `sanctions_clear`, or that filed the receipt as its screening record, held a signed statement that a wallet had been screened and cleared when no list was read.

No payment was made in the report. We have not demonstrated a payment released on the strength of the false verdict, and we are not claiming one was.

## Fix

1. A screening result now carries `screened`, true only when a list query completes.
2. When nothing was screened the endpoint returns `verdict: "flag"`, `sanctions_clear: null`, `sanctions_screened: false`, a reason that names the situation, and no receipt at all.
3. Unsupported networks no longer report `sanctions_clear: true`.
4. `/discovery/resources` derives `sanctions_screening` from the switch instead of hardcoding true.
5. The hosted recurring authority endpoint refuses to activate a standing payment authority when screening is enabled but no lookup ran, and records the gap on the audit chain either way.
6. `/compliance/attestation` lists only sources that have a configured feed.
7. The documentation pages and the public compliance hub no longer claim live screening.

## Timeline (UTC, 2026-11-25)

| Time | Event |
|---|---|
| 15:46 | Report received |
| 17:02 | Reproduced against production, and against OFAC SDN.XML published 23 November |
| 18:03 | Fix merged to the platform master branch |
| 18:42 | Screening enabled in production |
| 18:50 | Code hotfix deployed and verified |
| 19:12 | Rebuilt into the container image and recreated from compose |

## Verification

```
curl -s https://api.algovoi.co.uk/compliance/attestation
```

Expect `active_screening.sanctions_enabled: true` and `sanctions_sources` naming only configured feeds.

```
curl -s -X POST https://api.algovoi.co.uk/compliance/screen \
  -H 'Content-Type: application/json' \
  -d '{"recipient_address":"0x098B716B8Aaf21512996dC57EB0615e2383E2f96","network":"eip155:8453"}'
```

Expect `verdict: "block"`, `sanctions_clear: false`, `sanctions_screened: true`, and `compliance_receipt.screen_result: "DENY"`.

```
curl -s -X POST https://api.algovoi.co.uk/compliance/screen \
  -H 'Content-Type: application/json' \
  -d '{"recipient_address":"0x098B716B8Aaf21512996dC57EB0615e2383E2f96","network":"totally-bogus-chain"}'
```

Expect `verdict: "flag"`, `sanctions_clear: null`, `sanctions_screened: false`, and no receipt.

## Credit

Found and reported by kta1kri, https://github.com/kta1kri, while reviewing the public API with unauthenticated requests. The false clearance needed one request with a public SDN address, no payment made.