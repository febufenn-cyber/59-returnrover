# ReturnRover — Low-Level Design

## 1. Architecture

```
Customer (D2C brand's customer, no account)
   |  visits portal.returnrover.app?store=<brand-slug>
   |  enters order#+email, selects items, reason, uploads photos
   v
Cloudflare Worker (TypeScript, single Worker, /api/* routes)
   |
   +--> POST /api/returns -> validate order#+email against `orders` cache (IP
   |       rate-limited, public, no auth) -> creates return_request (submitted)
   +--> POST /api/returns/:id/photos -> private Storage bucket; on last photo,
   |       ctx.waitUntil(triggerTriage(id)) — non-blocking, instant response to customer
   +--> triggerTriage(): anthropic claude-sonnet-4-6 vision call (sync — Section 4),
   |       writes triage_results, status -> triaged
   +--> Merchant dashboard (JWT, brand-scoped via brand_users)
   |       GET /api/dashboard/returns, POST /api/dashboard/returns/:id/decide
   +--> POST /api/webhooks/orders <- Shopify/WooCommerce webhook (HMAC-verified),
           upserts `orders` — how lookup works with no installed app (Section 7)
```

## 2. Data model

Multi-tenant: every brand-scoped table carries `brand_id`; RLS checks membership via
`brand_users`, not raw `auth.uid()` ownership.

```sql
create table brands (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  platform text not null,           -- 'shopify' | 'woocommerce'
  webhook_secret text not null,     -- HMAC secret for verifying inbound webhooks
  credits int not null default 20,  -- monthly triage pool, reset on billing cycle
  created_at timestamptz not null default now()
);

create table brand_users (
  brand_id uuid not null references brands(id),
  user_id uuid not null references auth.users(id),
  primary key (brand_id, user_id)
);
alter table brand_users enable row level security;
create policy "own membership" on brand_users for select using (auth.uid() = user_id);

-- webhook-synced order cache (source of truth is the platform; this is a cache)
create table orders (
  id uuid primary key default gen_random_uuid(),
  brand_id uuid not null references brands(id),
  external_order_id text not null,
  customer_email text not null,
  order_data jsonb not null,
  synced_at timestamptz not null default now(),
  unique (brand_id, external_order_id)
);

-- state machine: submitted -> photos_uploaded -> triaged -> decided -> closed
create table return_requests (
  id uuid primary key default gen_random_uuid(),
  brand_id uuid not null references brands(id),
  order_id uuid not null references orders(id),
  customer_email text not null,
  items jsonb not null,
  reason text not null,
  status text not null default 'submitted',
  created_at timestamptz not null default now()
);

create table return_photos (
  id uuid primary key default gen_random_uuid(),
  return_request_id uuid not null references return_requests(id),
  storage_path text not null            -- private bucket, never public
);

create table triage_results (
  id uuid primary key default gen_random_uuid(),
  return_request_id uuid not null references return_requests(id),
  recommendation text not null,    -- 'refund' | 'replace' | 'deny'
  confidence numeric not null,
  reasoning text not null,
  model_version text not null,
  created_at timestamptz not null default now()
);

alter table return_requests enable row level security;
create policy "brand members read" on return_requests for select
  using (exists (select 1 from brand_users where brand_id = return_requests.brand_id
                 and user_id = auth.uid()));
-- inserts from the public /api/returns route go via the service-role key, not RLS
```

## 3. API routes

| Route | Method | Auth | Behavior | Failure modes |
|---|---|---|---|---|
| `/api/webhooks/orders` | POST | HMAC signature | Upserts `orders` cache from Shopify/WooCommerce | 401 bad signature |
| `/api/returns` | POST | public, IP rate-limited | Validate order#+email, create return_request | 404 order not found; 429 rate limit |
| `/api/returns/:id/photos` | POST | scoped token (returned from create) | Store photo, trigger triage on final upload | 400 bad file type/size |
| `/api/dashboard/returns` | GET | JWT (brand member) | List brand's returns, filterable by status | 500 on DB error |
| `/api/dashboard/returns/:id/decide` | POST | JWT (brand member) | Merchant approves/overrides recommendation, status -> decided | 403 not brand member |
| `/config.js` | GET | none | Public Supabase URL/anon key | — |

## 4. LLM strategy

Provider: anthropic `claude-sonnet-4-6`, vision, `output_config` strict JSON schema.
Input: up to 6 customer-submitted photos as image blocks + item/reason text.

```json
{
  "recommendation": "refund | replace | deny",
  "confidence": 0.0,
  "damage_detected": true,
  "matches_stated_reason": true,
  "reasoning": "string, shown to the merchant, not the customer"
}
```

**Sync vs async: sync, fired via `ctx.waitUntil` so it doesn't block the customer's
response.** A single triage call (a handful of images, short prompt) completes in low
single-digit seconds — nowhere near the ~100s Worker cap, so it does not need the VPS
async-callback pattern used for heavy/slow jobs elsewhere. It's decoupled from the
customer's request only so they aren't kept waiting on an LLM call they don't need
instantly; the merchant sees the recommendation moments later.

Cost estimate: ~1,500-3,000 input tokens (text + 4-6 compressed images) + ~250 output
tokens per triage — roughly Rs1-2 per call at current Claude Sonnet pricing. Rough
order-of-magnitude only; re-check actual pricing before sizing the Rs799/mo credit pool.

## 5. Frontend pages

- `/portal?store=<slug>` — order lookup (order#, email).
- `/portal/return` — item selection, reason, photo upload.
- `/portal/status` — return status tracker (public, token-linked).
- `/dashboard` — merchant login, return list with status filter.
- `/dashboard/returns/:id` — photos + LLM recommendation + approve/override.
- `/dashboard/settings` — webhook URL + secret to paste into Shopify/WooCommerce admin.

## 6. Error handling + credits/refund flow

Standard reference pattern: `spend_credit` (1 credit = 1 triage) before the vision
call; `refund_credit` on any LLM error (timeout, malformed JSON, provider error) —
the return still exists and is flagged "triage failed, needs manual review" so the
merchant can still act on it without having been charged.

## 7. Integrations and launch gates

- **Shopify: full embedded app + App Store listing needs Shopify's app review**
  (OAuth scope justification, mandatory GDPR webhooks, multi-week queue). v1
  sidesteps this entirely — no app install, no OAuth, no listing. The brand manually
  adds a webhook (Shopify Admin → Notifications → Webhooks) pointing at
  `/api/webhooks/orders`, using a shared secret entered in `/dashboard/settings`.
  This unblocks order lookup and return intake; it does NOT unblock automatic refund
  execution via Shopify's Admin API (see Out of scope).
- **WooCommerce: no formal review gate** — webhook is configured directly by the
  store owner in wp-admin. Same standalone-portal approach applies without a gate.
- Photo-evidence vision triage accuracy is inherently imperfect — treat this as an
  ongoing product risk, not a one-time caveat: never let it become an auto-action.

## 8. Security notes

- RLS on all brand-scoped tables via `brand_users` membership, not raw `auth.uid()`
  ownership (multi-tenant, not single-owner rows).
- Return photos in a private Storage bucket; signed URLs only, never public.
- Webhook HMAC verification against each brand's `webhook_secret` — reject unsigned
  or mismatched-signature payloads before writing to `orders`.
- `/api/returns` and `/api/returns/:id/photos` are unauthenticated by necessity
  (customers have no account) — rate-limit by IP and require the order#+email match
  before accepting a submission, to prevent spam and enumeration.
- File type/size validation on photo uploads before they reach Storage or the LLM.

## 9. Out of scope for v1

- Embedded Shopify App Store app (see Section 7 gate).
- Automatic refund execution via Shopify/WooCommerce Admin APIs — v1 records the
  decision in ReturnRover only; the merchant executes the actual refund in their own
  platform admin.
- Return shipping label generation.
- Multi-language customer portal.
- Analytics/reporting beyond the basic status-filtered list.
