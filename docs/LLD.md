# PincodeIQ — Low-Level Design

## 1. Architecture

```
API consumer (seller's backend / D2C cart or checkout logic)
   |  GET /api/v1/serviceability?pincode=560001&courier=all
   |  Header: Authorization: Bearer <api_key>
   v
Cloudflare Worker (TypeScript, single Worker, /api/* routes)
   1. hash(api_key) -> KV "apikey:<hash>" (fast path, TTL 5m)
        miss -> lookup Supabase api_keys table, populate KV
   2. quota check -> KV "quota:<key_hash>" counter (see Section 6: eventually
      consistent by design, bounded by nightly Postgres reconciliation)
   3. pincode lookup -> KV "pincode:<code>:<courier>" (TTL 24h, version-tagged)
        miss -> query Supabase Postgres, populate KV
   4. ctx.waitUntil(): async usage-log write to Supabase (billing source of truth)
   v
Response: { pincode, zone, couriers: [{name, serviceable, cod, delivery_days_min,
            delivery_days_max, confidence}], as_of }

Dashboard (Workers Assets, separate frontend, for sellers managing keys)
   |  supabase-js CDN magic-link auth -> Supabase GoTrue
   |  fetch /api/dashboard/*  (Authorization: Bearer <jwt>, NOT the API key)
   v
Worker /api/dashboard/* — JWT-authed, distinct auth path from the API-key routes above
```

Two separate auth paths on purpose: the metered `/api/v1/*` product routes use
long-lived API keys (what a seller's backend calls), while `/api/dashboard/*` uses
normal Supabase JWTs (what a human logs into to manage keys and view usage) — never
mix the two.

## 2. Data model

```sql
-- source-of-truth coverage data, keyed by pincode + courier
create table pincodes (
  pincode text primary key,        -- 6-digit, validated at insert
  city text, state text, district text, zone text
);

create table courier_coverage (
  id uuid primary key default gen_random_uuid(),
  pincode text not null references pincodes(pincode),
  courier_name text not null,
  serviceable boolean not null,
  cod_available boolean not null default false,
  delivery_days_min int,
  delivery_days_max int,
  confidence numeric not null default 0.5,   -- rises as corrections agree
  source text not null,                       -- 'india_post_seed' | 'user_correction'
  updated_at timestamptz not null default now(),
  unique (pincode, courier_name)
);
alter table courier_coverage enable row level security;
create policy "public read" on courier_coverage for select using (true);
-- writes only via service-role key (Worker) or the correction-review path below

-- user-contributed corrections: pending -> approved | rejected (manual review via
-- Supabase table editor in v1, no admin UI — see Out of scope)
create table corrections (
  id uuid primary key default gen_random_uuid(),
  submitted_by uuid references auth.users(id),
  pincode text not null,
  courier_name text not null,
  field text not null,             -- 'serviceable' | 'cod_available' | 'delivery_days'
  proposed_value text not null,
  status text not null default 'pending',  -- pending|approved|rejected
  created_at timestamptz not null default now()
);
alter table corrections enable row level security;
create policy "insert own" on corrections for insert with check (auth.uid() = submitted_by);
create policy "read own" on corrections for select using (auth.uid() = submitted_by);

-- API keys: hashed at rest, never store plaintext
create table api_keys (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id),
  key_hash text not null unique,   -- sha-256 of the key
  key_prefix text not null,        -- first 8 chars, for display ("pk_live_a1b2...")
  quota_limit int not null default 50000,
  revoked_at timestamptz
);
alter table api_keys enable row level security;
create policy "own keys" on api_keys for all using (auth.uid() = user_id);

-- billing source of truth (async-written, not on the hot path)
create table usage_daily (
  api_key_id uuid not null references api_keys(id),
  day date not null,
  call_count int not null default 0,
  primary key (api_key_id, day)
);
```

No multi-step state machine beyond `corrections.status` (pending → approved/rejected,
transitioned manually by Febin via Supabase table editor in v1).

## 3. API routes

| Route | Method | Auth | Behavior | Failure modes |
|---|---|---|---|---|
| `/api/v1/serviceability` | GET | API key | KV-cached pincode+courier lookup | 401 bad key; 404 unknown pincode; 429 quota exceeded |
| `/api/v1/corrections` | POST | JWT | Submit a correction (status=pending) | 400 invalid pincode/field |
| `/api/dashboard/keys` | GET, POST | JWT | List / create API keys (key shown once on create) | 500 on DB error |
| `/api/dashboard/keys/:id` | DELETE | JWT | Revoke a key | 404 not owned |
| `/api/dashboard/usage` | GET | JWT | Usage stats for current billing period | 500 on DB error |
| `/config.js` | GET | none | Public Supabase URL/anon key | — |

## 4. LLM strategy

None. This is a deliberate architecture decision, not deferred scope: PincodeIQ is a
lookup product against structured data, and adding an LLM call to the request hot path
would add latency and cost with no accuracy benefit over direct table lookups. No
provider, no prompt, no JSON schema — there is nothing to design here for v1.

## 5. Frontend pages

- `/` — public docs page: API reference, example request/response, pricing.
- `/dashboard` — API key list, create/revoke, usage graph (auth-gated).
- `/dashboard/corrections` — submit a correction for a pincode/courier (auth-gated).
- `/login` — magic-link form.

## 6. Error handling + quota flow

A successful (200) response consumes one call against the key's monthly quota;
401/404/429 responses do not. Quota is enforced via a KV counter incremented on every
successful call — **KV is eventually consistent across edge locations, so this is an
approximate limit, not a hard cap**; a burst of concurrent requests near the quota
boundary may allow a small overage (documented tolerance: up to ~2% of the monthly
limit). A nightly cron job reconciles `usage_daily` (written via `ctx.waitUntil`,
Postgres-accurate) against the KV counter and resets/corrects it, and hard-blocks any
key that is genuinely and persistently over quota after reconciliation. This trade-off
(fast approximate enforcement + accurate nightly settlement) is intentional — a
strictly atomic per-request counter would need Durable Objects, which is out of scope
for the "pure Workers+KV+Supabase" constraint on this product.

## 7. Integrations and launch gates

No third-party OAuth or API-partnership gate blocks this launch — unlike other
products in this batch, PincodeIQ's data starts from a public source (India Post's
published PIN code directory) plus first-party user corrections, not a gated private
API. The real launch risk is data coverage/quality at cold start, not integration
approval — mitigate by seeding with the full India Post pincode list before opening
signups, and by being explicit in the docs about confidence levels per record.

## 8. Security notes

- RLS on `api_keys`, `corrections`, `usage_daily`; `pincodes` and `courier_coverage`
  are public-read (no PII).
- API keys: generate as `pk_live_<32 random chars>`, store only `sha256(key)` and an
  8-char `key_prefix` for display; show the full key exactly once, at creation.
- Input validation: pincode must be a 6-digit numeric string matching a real Indian
  PIN code range before it reaches the DB or cache layer.
- Rate-limit unauthenticated routes (`/api/v1/corrections` still requires JWT; no
  public unauthenticated write path exists).
- Secrets (Supabase service-role key) via `wrangler secret put` only.

## 9. Out of scope for v1

- Admin review UI for corrections (reviewed manually via Supabase table editor).
- Per-courier official data partnerships (v1 is public-data + community-sourced).
- Durable-Objects-backed exact quota counting (KV approximation is accepted for v1).
- Webhooks / bulk CSV upload endpoints.
- Non-Indian pincodes / international serviceability.
