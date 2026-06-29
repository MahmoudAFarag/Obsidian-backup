

Answer to your question

The documented flow and features are still accurate — the backend changes were mostly around the documented surface, not changes to it. The order state machine, the payment-policy concept (placedPSP/deliveredPSP/nonRefundable, MAXTTOC/sett_MAXTTOC), the PSP structure, and all four worked examples still match the backend. But a handful of concrete API-contract details drifted, several of them breaking for a seller integrating against v0.7. I created an updated Seller Integration Document v0.8.docx (v0.7 left untouched) fixing them.

What I changed in v0.8

Breaking fixes:
- Auth — "Bearer Token" → send the key in a custom access_token request header (verified in api_key.guard.ts:51).
- totalOrderPrice → orderTotalPrice (request table + both example JSONs + the error-message example).
- subSellerID → sellerID (description kept).
- Added currency as a required field ("must be sent — determines the payment gateway"), inserted into the Basic Integration group + both examples.
- MAXTTOC flipped from Optional/default 0 → Required when sending a custom paymentPolicy.

Inaccuracy fixes:
- amountType enum now lists 'R' = REMAINING.
- Error section — removed the 401 claim; clarified missing/invalid token returns HTTP 400 with {status:1, message:"Invalid Token"}.

Kept as-is (per your call): the webhook/Callback URL section — documented as intended design for the team to implement later.

Verified

- Every edit re-extracted and confirmed; the request-body table's vertical-merge ("Basic Integration") stayed intact with currency as a continue-row; file is a valid .docx.

Two things flagged for the backend team (not doc changes)

1. currency: the doc now says required, but code defaults it (seller_actions_handeler.ts:139) and currently picks the gateway from the delivery company's currency (lines 145-146) — worth aligning.
2. Direct-seller webhook: not built yet — only Zid/Salla marketplace adapters receive state changes; no callbackUrl is stored and no POST fires for Direct sellers.

I also saved a few project notes to memory (canonical backend = main_top_server @ scalable, the currency decision, and the webhook status) for future doc-vs-code work.

One thing I did not do, since you scoped it out: documenting the new surrounding features (service fees/TOF, GET /childSellers, GET /paymentPolicies/:sellerID, seller hierarchy, Disputed/Resolved states). Let me know if you'd like those added in a later pass.