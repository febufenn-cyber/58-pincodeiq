# PincodeIQ

A serviceability API: given a pincode, tells D2C sellers which couriers cover it, COD
availability, delivery zone, and estimated delivery days.

**Status: planned — not yet built (50-SaaS challenge #58)**

## The problem

D2C sellers integrating checkout need to know, per pincode, whether a courier will
actually deliver there, whether COD is available, and roughly how long it'll take —
before the customer completes an order. This data is scattered across each courier's
own tools and is rarely available as a clean, fast API a seller's own backend can call.

## Target buyer

D2C tooling builders and sellers who want this as an API call inside their own
checkout/cart logic — not a dashboard for humans to click through, an endpoint for
their backend to hit.

## Pricing hypothesis

Rs999/mo per 50,000 API calls, metered, tiered above that.

## Stack summary

- Cloudflare Worker (TypeScript, ESM) — API-first: KV-cached hot path, no LLM anywhere
  in the request path.
- Supabase Postgres (RLS) as the source of truth for pincode/courier coverage data and
  user-contributed corrections; KV is a cache in front of it, not the record of truth.
- No LLM. This is a lookup product, not a generation product.

## Risks / constraints (do not soften — read before building)

- **Data provenance is thin at launch.** v1 coverage starts from India Post's publicly
  published PIN code directory plus user-contributed corrections — not an authoritative,
  live feed from every courier. Accuracy improves over time as corrections come in;
  say so honestly in the product's own docs, do not imply real-time courier-verified
  data on day one.
- **No LLM in the hot path** — this is a deliberate architecture choice, not a
  cost-cutting shortcut to revisit later. Keep it pure Workers+KV+Supabase.
- KV-based quota counting is eventually consistent, not atomic — see `docs/LLD.md`
  Section 6 for the accepted overage tolerance and the Postgres reconciliation job that
  bounds it.

## How to continue this build

Read `docs/LLD.md` for the architecture, data model, and API contract, then
`docs/PLAN.md` for the ordered build tasks, and `CLAUDE.md` for repo conventions.
