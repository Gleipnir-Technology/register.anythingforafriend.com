# AGENTS.md — Anything for a Friend 5K Registration

## Project Purpose

A multi-step charity 5K registration web application. Runners select a beneficiary, enter their info, pay ($35 + $2.50 processing + optional donation), and receive a confirmation. The site is deployed on a single NixOS server behind Caddy.

## Tech Stack

| Layer | Technology | Notes |
|-------|-----------|-------|
| Runtime | **NixOS** | `flake.nix` provides dev shell and deployment |
| Backend | **Go 1.22+** | stdlib `net/http` or `chi` router |
| Database | **PostgreSQL 16** | Migrations as raw SQL files |
| DB Layer | **pgx v5** (pool) + **go-jet** (type-safe query builder) | jet generates types from live DB schema |
| Frontend | **Vue 3** + **TypeScript** | Vite, composition API, `<script setup>` |
| State | **Pinia** | Single store holds multi-step registration form state |
| CSS | **Bootstrap 5.3** + **Bootstrap Icons** | (as used in mockups) |
| Maps | **MapLibre GL JS 3.6** | Used on confirmation page for race route |
| Reverse Proxy | **Caddy** | TLS termination, `/api/*` proxy, static file serving |
| Payment | **Stripe** (deferred) | Placeholder form in mockups; real Stripe Elements later |

## Project Directory Layout

```
/workspace/register.anythingforafriend.com/
├── PLAN.md                  # Full implementation plan (phases, tests)
├── AGENTS.md                # This file — agent context
├── flake.nix                # Nix dev shell & deployment
├── .gitignore
├── backend/
│   ├── go.mod / go.sum
│   ├── cmd/server/main.go   # Entry point
│   ├── internal/
│   │   ├── config/          # Env-based config
│   │   ├── handler/         # HTTP handlers
│   │   ├── model/           # go-jet generated types (do not edit)
│   │   ├── server/          # Router setup, middleware
│   │   └── store/           # pgx pool setup, query functions
│   └── migrations/          # *.up.sql / *.down.sql
├── frontend/
│   ├── index.html           # Vite entry
│   ├── vite.config.ts
│   ├── tsconfig.json
│   ├── package.json
│   └── src/
│       ├── main.ts          # createApp + router + pinia
│       ├── App.vue          # Shell (nav, router-view, footer)
│       ├── router/index.ts
│       ├── stores/          # Pinia stores
│       │   └── registration.ts  # Multi-step form state
│       ├── api/             # Typed axios client
│       ├── views/           # Page components (one per step)
│       │   ├── HomePage.vue
│       │   ├── BeneficiaryPage.vue
│       │   ├── RegisterPage.vue
│       │   ├── PaymentPage.vue
│       │   └── CompletePage.vue
│       ├── components/      # Shared UI components
│       ├── composables/     # useApi, useFormValidation, etc.
│       └── types/           # TypeScript interfaces
├── caddy/
│   └── Caddyfile
└── static/                  # Original mockup images (migrate to frontend/public/)
    └── images/              # afaf-logo.png, team avatars, etc.
```

## Database Schema (from mockups)

Four tables, see `backend/migrations/001_initial_schema.up.sql` for exact DDL:

- **beneficiaries** — id, name, description, team_color (hex), avatar_url, team_lead_name, team_lead_phone, team_lead_email
- **registrations** — id, beneficiary_id (FK), full_name, email, phone, can_receive_texts (bool), emergency_name, emergency_phone, emergency_relationship, tshirt_size, waiver_requested (bool), waiver_reason, payment_status, created_at
- **child_registrations** — id, registration_id (FK), name, tshirt_size
- **payments** — id, registration_id (FK), amount_base, amount_processing, amount_donation, total_amount, stripe_payment_intent_id, status, created_at

## Seed Data (6 beneficiaries from mockups)

| Name | Color | Description |
|------|-------|-------------|
| Sarah Johnson | #FF6B6B (Sunset Red) | Medical expenses for cancer treatment |
| Michael Chen | #4ECDC4 (Ocean Teal) | Recovery support after accident |
| Emma Rodriguez | #FFD93D (Sunshine Yellow) | Support for family during hardship |
| David Thompson | #A8E6CF (Mint Green) | Rehabilitation after injury |
| Lisa Patel | #B39DDB (Lavender Purple) | Educational fund for children |
| James Wilson | #FF8C94 (Coral Pink) | Home modifications for accessibility |

*(Avatar images currently use Unsplash URLs in mockups; will be moved to local static assets.)*

## Design Tokens

```css
--primary-color: #FFD1ED;   /* light pink, used in gradients */
--secondary-color: #580C28; /* deep maroon, used for text, borders, accents */
```

The hero section on each page uses `linear-gradient(135deg, var(--primary-color) 0%, var(--secondary-color) 100%)`. Progress dots and CTAs use the same gradient.

## API Endpoints (Planned)

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/health` | Health check (DB ping included) |
| GET | `/api/beneficiaries` | List all beneficiaries |
| POST | `/api/beneficiaries` | Create new beneficiary (from Step 1 "add new") |
| POST | `/api/registrations` | Submit registration (Step 2 form data) |
| POST | `/api/payments` | Record payment (Step 3, placeholder until Stripe) |
| GET | `/api/registrations/:id` | Full registration detail (Step 4 confirmation) |

All request/response bodies are JSON. Use `Content-Type: application/json`.

## Registration Form Data Shape (Step 2 → POST /api/registrations)

```json
{
  "beneficiary_id": "uuid",
  "full_name": "Jane Runner",
  "email": "jane@example.com",
  "phone": "(555) 123-4567",
  "can_receive_texts": true,
  "emergency_name": "John Runner",
  "emergency_phone": "(555) 987-6543",
  "emergency_relationship": "spouse",
  "tshirt_size": "m",
  "waiver_requested": false,
  "waiver_reason": "",
  "children": [
    { "name": "Timmy Runner", "tshirt_size": "youth-m" }
  ]
}
```

### Validation Rules (from mockup JavaScript)

- **Full name**: at least 2 words, min 2 chars total
- **Email**: must match `/[^\s@]+@[^\s@]+\.[^\s@]+/`
- **Phone**: exactly 10 digits (stripped); UI formats as `(555) 123-4567`
- **T-shirt size**: one of `youth-s`, `youth-m`, `youth-l`, `xs`, `s`, `m`, `l`, `xl`, `2xl`, `3xl`
- **Child registrations**: each child needs name + t-shirt size (both required if any children added)
- **Waiver**: if requested, reason textarea is visible but not required

## Page Flow & Routes

```
/                          → HomePage (landing)
/register/beneficiary      → BeneficiaryPage (Step 1: pick team)
/register/info             → RegisterPage (Step 2: your info)
/register/payment          → PaymentPage (Step 3: pay)
/register/complete/:id     → CompletePage (Step 4: confirmation)
```

The original mockups used `.html` file links. The SPA uses Vue Router with history mode. Caddy `try_files {path} /index.html` handles client-side routing.

## How to Start Developing

```bash
# 1. Enter dev shell
nix develop

# 2. Start Postgres & create database
pg_ctl -D /tmp/pgdata start
createdb afaf_registration

# 3. Run migrations
psql afaf_registration < backend/migrations/001_initial_schema.up.sql
psql afaf_registration < backend/migrations/002_seed_beneficiaries.up.sql

# 4. Start backend (terminal 1)
cd backend
DATABASE_URL="postgres:///afaf_registration?host=/tmp" go run ./cmd/server

# 5. Start frontend dev server (terminal 2)
cd frontend
npm run dev
# Vue dev server at http://localhost:5173 proxies /api to :8080

# 6. (For full stack integration testing, also start Caddy)
caddy run --config caddy/Caddyfile
```

## Testing

- **Backend unit/integration**: `cd backend && go test ./...`
- **Frontend unit**: `cd frontend && npx vitest run`
- **Frontend type-check**: `cd frontend && npx vue-tsc --noEmit`
- **End-to-end (local)**: Start all services, then walk through `localhost:5173` → beneficiary → register → payment → complete

## Key Conventions

1. **Go package layout**: `cmd/` for binaries, `internal/` for application code (not importable externally), `internal/handler/` for HTTP handlers, `internal/store/` for database queries.
2. **go-jet**: Types in `internal/model/` are **generated** — never edit them by hand. Re-run jet after schema changes.
3. **Vue components**: Use `<script setup lang="ts">` with composition API. One component = one file. Extract reusable pieces into `components/`.
4. **Form validation**: Validate both client-side (Vue) and server-side (Go). The Go API must enforce all rules independently.
5. **Multi-step form state**: Stored in Pinia. The store persists during the session (lost on refresh — acceptable for this flow).
6. **API base URL**: In dev, Vite proxies `/api` to `http://localhost:8080`. In production, Caddy routes `/api/*` to the Go backend. The frontend always calls `/api/*` (relative URL).
7. **Error handling**: Go handlers return `{"error": "message"}` with appropriate HTTP status codes. Vue shows errors inline on form fields or as toast notifications.

## What's Already Done

- Mockup HTML pages for all 5 screens exist in the git history (not yet checked out as working files)
- Static images in `static/images/` (logo, background, team avatars placeholder)
- MIT License

## What Needs to Be Built (Order)

1. `flake.nix` — dev shell
2. Database migrations + seed data
3. Go backend: config → pool → jet generation → health endpoint → API endpoints
4. Vue frontend: Vite project → router → Pinia store → API client → page components
5. Caddy integration
6. NixOS deployment module

See `PLAN.md` for the detailed phase-by-phase breakdown with test checkpoints.

## Deferred / Not Yet Decided

- **Stripe integration**: Payment form is a placeholder. Will add `@stripe/stripe-js` + Stripe Elements later. Backend needs `github.com/stripe/stripe-go` for PaymentIntent creation and webhook handling.
- **Authentication**: No login needed for registration flow. Admin dashboard would need auth.
- **Email sending**: Confirmation emails are referenced but not implemented.
- **Race date**: Currently hardcoded as "June 15, 2024" in mockups. Make configurable.
