# FarmIt v1 · Farmer-to-Consumer Marketplace

FarmIt is an Android-ready Next.js prototype of a geofenced farmer-to-consumer marketplace: farmers list Sona Masuri paddy lots, and consumers inside a 100 km sourcing ring around Jaynagar, Bengaluru buy one transparent 20 kg weekly order from the nearest farms. Built for SIH 2026 problem statement 26033.

## Run locally

```bash
corepack pnpm install
corepack pnpm dev
```

Open `http://localhost:3000`. Sign in with one of the seeded demo accounts to exercise the complete farmer → operator → consumer path without a remote Supabase project. Sign out and switch accounts between stages. The operator generates the server-owned quote snapshot per farm lot; the consumer can then reserve or decline it.

If the catalog API is unavailable, the consumer screen automatically falls back to local seeded demo lots and a local price/escrow preview so the frontend remains clickable before a real backend or database is connected. Live listing, quote publication, and persistence still require the API routes.

The production build and typecheck can be run with `corepack pnpm build` and `corepack pnpm typecheck`.

## Authentication

The prototype is gated by a basic email + password sign-in (`POST /api/auth/login`, `/login`). Sessions are stateless HMAC-signed tokens in an httpOnly cookie (`farmit_session`, expiry 7 days). Edge middleware enforces the gate — unauthenticated page requests redirect to `/login` (preserving the destination via `?next=`), and unauthenticated API calls get `401`; the `/api/auth/*` endpoints are always public. Sign out via `POST /api/auth/logout`.

Seeded demo accounts (also shown on the login page, one-click):

| Email | Password | Role |
|---|---|---|
| `consumer@farmit.in` | `consumer123` | consumer — shop the 20 kg weekly order, doorstep QR handoff |
| `farmer@farmit.in` | `farmer123` | farmer — list harvest lots in the catalog |
| `operator@farmit.in` | `operator123` | operator — publish quote snapshots, plan logistics |

Passwords are stored as scrypt hashes with per-user salts in `apps/web/lib/users.ts` (demo seed; Supabase `auth.users` replaces it at pilot). Set `AUTH_SECRET` to a long random value in deployed environments — the code falls back to a dev-only demo secret so the seed works out of the box.

## Documentation

| Doc | Read it for |
|---|---|
| [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) | System design, flows, pricing model, demo↔production boundary |
| [`docs/ROADMAP.md`](docs/ROADMAP.md) | Phase plan and PS-26033 requirements matrix |
| [`docs/DEPLOYMENT.md`](docs/DEPLOYMENT.md) | Pilot/production environments, Supabase wiring, scaling notes |
| [`docs/phases/`](docs/phases) | Per-phase logs: what was implemented, why, test evidence, gaps |
| [`docs/decisions/`](docs/decisions) | Architecture decision records (ADRs) |
| [`CHANGELOG.md`](CHANGELOG.md) | Version history |
| [`CONTRIBUTING.md`](CONTRIBUTING.md) | Setup, conventions, documentation contract, PR checklist |

## Workspace shape

- `apps/web`: Next.js PWA shell, demo workflows, route handlers, and the seeded lot catalog (`lib/catalog.ts`)
- `apps/web/lib/catalog.ts`: multi-farmer lot catalog (demo in-memory; retained across Next dev hot reloads, Supabase replaces it at pilot)
- `packages/domain`: shared records and role/status contracts
- `packages/pricing`: server-side yield conversion, rupee rounding, quote calculation, and 100 km haversine geofence
- `packages/forecast`: explainable weekly demand forecasting (weighted MA + trend + seasonal index)
- `packages/logistics`: FPO-hub relay route planning (nearest-neighbour + 2-opt, food-miles-saved metric)
- `packages/validation`: MSP floor payout, minimum-lot, and coordinate guards
- `packages/translations`: English/Kannada UI copy
- `packages/api-client`: typed fetch helpers for the marketplace endpoints
- `supabase/migrations`: PostgreSQL tables and row-level policies for the real auth-backed environment

The current API route handlers are intentionally demo-local. Before a pilot, replace the seeded lot catalog and in-memory order behavior with Supabase queries and email/password sessions (see `docs/DEPLOYMENT.md`). The quote snapshot should remain immutable after publication.

## Marketplace API

- `GET /api/lots?lat=…&lng=…` — server-side geofence scan. Returns every catalog lot annotated with `distanceKm` and `withinGeofence` (100 km radius, nearest first) plus `outsideCount` for lots the food-miles guardrail hides. Without query params it scans from the Jayanagar hub.
- `POST /api/lots` — farmer-only lot listing. Validates the MSP floor (₹24.41/kg), the 29.85 kg minimum (20 kg rice at 67% yield), and coordinates server-side.
- `GET/POST /api/marketplace/orders` — consumer order requests linked to a selected farmer lot. A request records rice quantity, delivery preference/notes, and a `requested` status for the operator workflow.
- `GET/POST /api/quote` — consumers read the latest published snapshot for a `lotId`; operators publish or replace it. The lot is resolved from the server-side catalog, so clients cannot spoof farmer payouts; unknown or expired snapshots cannot be reserved.
- `GET /api/escrow` / `POST /api/escrow` — consumer-only escrow lifecycle (`hold` | `dispatch` | `release`) keyed to published snapshots; release requires the doorstep delivery code.
- `GET /api/escrow/qr?code=…` — server-rendered SVG QR for the doorstep handshake.
- `GET /api/forecast?horizon=1..4` — explainable weekly demand forecast (history, point + confidence band, method metadata) plus the geofenced supply check that powers the consumer nudge.
- `POST /api/routes` — consolidated FPO-hub relay plan for open orders (farm-gate pickup tours + one bulk line-haul per hub) with consolidated-vs-individual food-miles metrics; accepts an explicit `orders[]` override.

Operator GST is entered in the UI as a percentage (for example, `5` means 5%); the API and pricing engine store it as a decimal fraction (`0.05`).

## Pricing guardrails

The demo uses a 67% documented milling yield and a ₹24.41/kg 2026–27 common-paddy MSP reference. A 20 kg rice offer therefore requires 29.85 kg of paddy before milling, packaging, logistics, platform charge, or tax. All visible seed figures are labelled illustrative demo data in the UI.

## Pricing guardrails

The demo uses a 67% documented milling yield and a ₹24.41/kg 2026–27 common-paddy MSP reference. A 20 kg rice offer therefore requires 29.85 kg of paddy before milling, packaging, logistics, platform charge, or tax. All visible seed figures are labelled illustrative demo data in the UI.