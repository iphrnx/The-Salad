# Deployment configuration

The Next.js app builds for Cloudflare Workers with OpenNext. Supabase provides authentication, Postgres, Storage, Realtime, and the Edge Functions. Use only the Supabase project assigned to this repository; do not reuse another project's URL or keys.

## App build

`npm ci` installs the locked dependencies. `npm run typecheck`, `npm run lint`, and `npm run build` are local checks; the build does not deploy. The Worker configuration is in `wrangler.jsonc` and `open-next.config.ts`. `npm run build:next` builds the Next.js standalone output without the Worker adapter.

| App variable | Purpose |
|---|---|
| `NEXT_PUBLIC_SUPABASE_URL` | Intended Supabase project URL |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Public anon or publishable key for that project |
| `SUPABASE_SERVICE_ROLE_KEY` | Server-only key; never expose it through `NEXT_PUBLIC_*` |
| `NEXT_PUBLIC_SITE_URL` | App origin used for metadata and links |
| `NEXT_PUBLIC_EDGE_BASE` | Optional local Edge Function base URL override |

Set Supabase Auth's Site URL and allowed redirects to the intended app origin and approved development origins. Browser requests to Edge Functions need matching CORS origins.

## Edge Functions

The deployment helpers in `scripts/` require an explicit `SUPABASE_PROJECT_ID`. They do not infer a target from the frontend URL or use a built-in project. If the Supabase MCP deploy tool is namespaced, set `MCP_DEPLOY_TOOL` to its exact callable name. Confirm the project and tool before any future deployment.

The bundled handlers apply their own request authentication where needed, including staff PIN sessions, diner session tokens, and signed payment webhooks. Preserve the existing gateway JWT setting when deploying these handlers.

| Edge Function configuration | Purpose |
|---|---|
| `ALLOWED_ORIGINS` / `SITE_URL` | Allowed browser origins |
| `GEMINI_API_KEY` | AI menu extraction and image generation |
| `RAZORPAY_KEY_ID`, `RAZORPAY_KEY_SECRET`, `RAZORPAY_WEBHOOK_SECRET` | Subscription and signed webhook integration |

Supabase supplies its URL and API keys to Edge Functions. Keep private integration credentials in the target environment's secret store. The Razorpay webhook entry point is `/functions/v1/razorpay-webhook` on the intended project's URL.

## Database

Phase 1 retains `supabase/migrations/` unchanged as the authoritative schema reference for the next phase. Do not apply migrations or deploy from this cleanup phase.
