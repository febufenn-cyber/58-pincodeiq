# PincodeIQ — Build Plan

Ordered tasks, TDD style. Each is sized for one focused agent session.

## T1 — Repo scaffold + Worker skeleton

Files: `wrangler.toml`, `src/index.ts`, `package.json`, `tsconfig.json` (strict).
Interface: `fetch(request: Request, env: Env): Promise<Response>` router stub.
Test first: `test/worker.test.ts` — unknown route returns 404, `/config.js` returns
Supabase URL/anon key.
**Done when:** `wrangler deploy --dry-run` succeeds.

## T2 — Supabase schema + RLS

Files: `supabase/migrations/0001_init.sql` (pincodes, courier_coverage, corrections,
api_keys, usage_daily from LLD Section 2).
Test first: migration-apply script asserting RLS blocks cross-user reads on
`api_keys` and `corrections` with two test users.
**Done when:** migration applies cleanly and the isolation test passes.

## T3 — India Post pincode seed data

Files: `supabase/seed/pincodes.sql` generated from the public India Post PIN code
directory.
Test first: seed script asserts row count matches expected total and every pincode is
a valid 6-digit string.
**Done when:** `pincodes` table is fully seeded on a fresh project.

## T4 — API key issuance + hashing

Files: `src/apiKeys.ts` — `generateApiKey(): { key: string; hash: string; prefix:
string }`, `hashApiKey(key: string): string` (SHA-256), `verifyApiKey(key: string,
env: Env): Promise<ApiKeyRecord | null>`.
Test first: assert `hashApiKey` is deterministic, `verifyApiKey` rejects a revoked key
and an unknown key, and the full key is never persisted anywhere.
**Done when:** unit tests pass against a mocked Supabase fetch.

## T5 — KV cache layer for pincode lookups

Files: `src/cache.ts` — `getCachedCoverage(pincode: string, env: Env): Promise<
CoverageResult | null>`, `setCachedCoverage(pincode: string, data: CoverageResult,
env: Env): Promise<void>` with a version tag for invalidation.
Test first: mock KV, assert a cache miss falls through to the Postgres query function
and populates the cache; assert TTL is set to 24h.
**Done when:** cache-then-DB fallback logic is covered by tests, no live KV needed.

## T6 — Quota counter (KV, eventually-consistent) + reconciliation job

Files: `src/quota.ts` — `checkAndIncrementQuota(keyHash: string, env: Env): Promise<
{ allowed: boolean; remaining: number }>`; `scripts/reconcile-quota.ts` (cron target)
comparing `usage_daily` totals against KV counters and hard-blocking genuinely
over-quota keys.
Test first: assert `checkAndIncrementQuota` denies once the KV counter reaches the
key's `quota_limit`; assert the reconciliation script flags a key whose Postgres total
exceeds its limit even if KV had drifted low.
**Done when:** both are unit-tested against mocks; document the ~2% overage tolerance
inline in code comments per LLD Section 6.

## T7 — Serviceability route

Files: `src/routes/serviceability.ts`.
Interface: `handleServiceability(req: Request, env: Env, ctx: ExecutionContext):
Promise<Response>` — validates pincode format, checks API key + quota, reads
cache-then-DB, logs usage via `ctx.waitUntil`.
Test first: mock all dependencies; assert 401 on bad key, 429 on quota exceeded, 404
on unknown pincode, 200 with correct JSON shape on success; assert usage log is
fire-and-forget (response doesn't await it).
**Done when:** all cases pass against mocks.

## T8 — Corrections route

Files: `src/routes/corrections.ts`.
Interface: `handleSubmitCorrection(req: Request, env: Env): Promise<Response>`.
Test first: assert 400 on invalid field/pincode, 200 + row inserted with
`status='pending'` on valid submission, RLS prevents reading others' corrections.
**Done when:** route tests pass against mocks.

## T9 — Dashboard routes + frontend

Files: `src/routes/dashboard.ts` (keys CRUD, usage stats), `public/index.html`
(docs+pricing), `public/dashboard.html`, `public/login.html`.
Test first: assert key creation returns the plaintext key exactly once and never
again on subsequent list calls (only `key_prefix` is returned thereafter).
**Done when:** a logged-in user can create a key, see it once, and see only the
prefix afterward.

## T10 — Deploy + live smoke test + launch checklist

Files: `scripts/smoke.ts`, `wrangler secret put` for Supabase service-role key.
Test: smoke script creates a test API key via the dashboard flow, calls
`/api/v1/serviceability` against a known seeded pincode, asserts correct shape and
that quota decremented by exactly 1.
Launch checklist: pricing page live ("Rs999/mo per 50k calls"), data-confidence
disclaimer visible in docs ("coverage improves via community corrections, not a live
courier feed"), secrets set via `wrangler secret put`, reconciliation cron scheduled.
**Done when:** smoke script passes against the production URL.
