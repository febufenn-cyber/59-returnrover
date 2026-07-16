# ReturnRover

A self-serve returns portal for D2C brands: customers submit a return with photos,
an LLM triages it (refund / replace / deny) as a recommendation, the merchant approves.

**Status: planned — not yet built (50-SaaS challenge #59)**

## The problem

D2C brands handle returns manually over email/WhatsApp — slow for customers, tedious
for support staff who have to eyeball photos and make a judgment call every time.
ReturnRover gives customers a self-serve flow and gives merchants an AI-drafted
recommendation to approve or override, instead of starting from a blank decision.

## Target buyer

Indian D2C brands running on Shopify or WooCommerce who currently handle returns by
hand and want the volume off their support queue without losing control of the
refund decision.

## Pricing hypothesis

Rs799/mo, including a monthly pool of AI-triaged returns.

## Stack summary

- Cloudflare Worker (TypeScript, ESM) — standalone portal + dashboard, no embedded
  Shopify/WooCommerce app in v1.
- Supabase: magic-link auth for merchants, Postgres+RLS (brand-scoped, multi-tenant),
  private Storage for return photos.
- LLM: anthropic claude-sonnet-4-6 vision for photo-based triage, recommendation only.

## Risks / constraints (do not soften — read before building)

- **Shopify App Store review is a launch gate for any embedded/listed app** — OAuth
  scope justification, GDPR webhook requirements, and a multi-week review queue.
  v1 deliberately avoids this: a standalone portal the brand links to, fed by a
  manually configured order webhook, with no app install and no App Store listing.
  See `docs/LLD.md` for exactly what that unblocks and what it defers.
- **LLM vision triage is a recommendation, never an auto-action.** Photo-based
  damage/authenticity assessment is imperfect; the merchant always makes the final
  refund/replace/deny call. ReturnRover never executes a refund automatically.
- WooCommerce has no equivalent formal review gate (self-hosted, webhook configured
  directly by the store owner) — same standalone approach works there without a gate.

## How to continue this build

Read `docs/LLD.md` for the architecture, data model, and LLM strategy, then
`docs/PLAN.md` for the ordered build tasks, and `CLAUDE.md` for repo conventions.
