# ReturnRover — Build Plan

Ordered tasks, TDD style. Each is sized for one focused agent session.

## T1 — Repo scaffold + Worker skeleton

Files: `wrangler.toml`, `src/index.ts`, `package.json`, `tsconfig.json` (strict).
Interface: `fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<
Response>` router stub.
Test first: `test/worker.test.ts` — unknown route 404, `/config.js` serves config.
**Done when:** `wrangler deploy --dry-run` succeeds.

## T2 — Supabase schema + multi-tenant RLS

Files: `supabase/migrations/0001_init.sql` (brands, brand_users, orders,
return_requests, return_photos, triage_results from LLD Section 2).
Test first: two brands, two users each in one brand only — assert a user cannot
read the other brand's `return_requests` via RLS.
**Done when:** migration applies and the cross-tenant isolation test passes.

## T3 — Webhook receiver (order sync)

Files: `src/routes/webhooks.ts` — `handleOrderWebhook(req: Request, env: Env):
Promise<Response>`, `verifyWebhookSignature(payload: string, signature: string,
secret: string): boolean`.
Test first: assert a request with a bad/missing HMAC signature is rejected with 401
before touching the DB; assert a valid payload upserts into `orders`.
**Done when:** signature verification and upsert logic pass against mocks.

## T4 — Return intake (public, rate-limited)

Files: `src/routes/returns.ts` — `handleCreateReturn(req: Request, env: Env):
Promise<Response>` validates order#+email against `orders`, creates a
`return_request`, returns a scoped upload token.
Test first: assert 404 when order#+email don't match any cached order; assert a
second rapid request from the same IP is rate-limited (429).
**Done when:** both cases pass against mocked KV rate-limiter and Supabase.

## T5 — Photo upload + triage trigger

Files: `src/routes/photos.ts` — `handleUploadPhoto(req: Request, env: Env, ctx:
ExecutionContext): Promise<Response>`, `triggerTriage(returnRequestId: string, env:
Env): Promise<void>`.
Test first: assert invalid file type/size is rejected (400); assert
`ctx.waitUntil` is called with `triggerTriage` on upload and the HTTP response does
not await it (response resolves before the triage promise settles in the test).
**Done when:** upload path and non-blocking trigger are both verified against mocks.

## T6 — Vision triage call

Files: `src/llm/triage.ts` — `runTriage(photos: string[], reason: string, items:
unknown): Promise<TriageResult>` calling anthropic claude-sonnet-4-6 with the JSON
schema from LLD Section 4.
Test first: mock the Anthropic fetch; assert a well-formed response parses into
`TriageResult`; assert a malformed/error response throws (caller refunds credit).
**Done when:** parsing and error paths are both unit-tested.

## T7 — Credits + refund wiring around triage

Files: `src/credits.ts` reused from reference pattern; wire `spend_credit` before
`runTriage` and `refund_credit` on any thrown error, updating `return_requests.status`
to `triage_failed` on failure instead of leaving it stuck at `photos_uploaded`.
Test first: assert credit is refunded and status set to `triage_failed` when
`runTriage` throws; assert credit is spent (not refunded) on success.
**Done when:** both paths pass against mocks.

## T8 — Merchant dashboard routes

Files: `src/routes/dashboard.ts` — `handleListReturns`, `handleDecide(req: Request,
env: Env): Promise<Response>` (approve/override/deny, brand-membership checked).
Test first: assert 403 when the JWT's user isn't in `brand_users` for the target
brand's return_request; assert a valid decide call updates status to `decided`.
**Done when:** both cases pass against mocks.

## T9 — Frontend pages

Files: `public/portal.html`, `public/return.html`, `public/status.html`,
`public/dashboard.html`, `public/settings.html`, `public/login.html`.
Test first: none (manual smoke) — but unit-test that `/dashboard/settings` renders
the brand's actual `webhook_secret` only to an authenticated brand member.
**Done when:** a full customer flow (lookup -> submit -> photo -> status) and a
merchant flow (login -> see triage -> decide) work manually end to end.

## T10 — Deploy + live smoke test + launch checklist

Files: `scripts/smoke.ts`, `wrangler secret put` for Supabase service-role key and
Anthropic API key.
Test: smoke script simulates a webhook order sync, submits a return with a test
photo, polls until `triaged`, and confirms the dashboard decide endpoint works.
Launch checklist: pricing page live, "AI recommendation only, not an automatic
refund" disclaimer visible in the merchant dashboard, webhook setup instructions
published, secrets set via `wrangler secret put`.
**Done when:** smoke script passes against the production URL.
