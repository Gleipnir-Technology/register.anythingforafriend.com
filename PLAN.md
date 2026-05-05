# Implementation Plan: Anything for a Friend 5K Registration

## Project Overview

A multi-step registration flow for a charity 5K run. Users progress through four steps:
1. **Choose Your Team** (beneficiary selection)
2. **Your Information** (contact info, t-shirt size, child registrations, waiver request)
3. **Payment** ($35 registration + $2.50 processing + optional donation)
4. **Registration Complete** (confirmation, race details, map, social sharing)

## Architecture

```
┌──────────┐     ┌───────┐     ┌──────────┐     ┌────────────┐
│  Browser │────▶│ Caddy │────▶│ Go App   │────▶│ PostgreSQL │
│  (Vue 3) │     │ :443  │     │ :8080    │     │ :5432      │
└──────────┘     └───────┘     └──────────┘     └────────────┘
                      │
                      ├── /api/*  → reverse proxy to Go
                      ├── /assets/* → static Vue build
                      └── TLS termination (LetsEncrypt/self-signed)
```

## Directory Layout (Target)

```
/workspace/register.anythingforafriend.com/
├── PLAN.md                          # This document
├── flake.nix                        # NixOS flake for devshell & deployment
├── backend/
│   ├── go.mod / go.sum
│   ├── cmd/server/main.go           # Entry point, server bootstrap
│   ├── internal/
│   │   ├── server/server.go         # HTTP server setup, routes
│   │   ├── handler/                 # HTTP handlers (per resource)
│   │   │   ├── registration.go
│   │   │   ├── beneficiary.go
│   │   │   └── health.go
│   │   ├── model/                   # jet-generated model types
│   │   ├── store/                   # database access layer (pgx + go-jet)
│   │   └── config/config.go         # Env-loaded config
│   ├── migrations/                  # SQL migration files
│   └── Dockerfile                   # (optional, for non-NixOS dev)
├── frontend/
│   ├── package.json
│   ├── vite.config.ts
│   ├── tsconfig.json
│   ├── index.html
│   ├── src/
│   │   ├── main.ts                  # Vue app bootstrap
│   │   ├── App.vue                  # Root component w/ router-view
│   │   ├── router/index.ts          # Vue Router routes
│   │   ├── views/                   # Page-level components
│   │   │   ├── HomePage.vue         # (index.html)
│   │   │   ├── BeneficiaryPage.vue  # Step 1 (beneficiary.html)
│   │   │   ├── RegisterPage.vue     # Step 2 (register.html)
│   │   │   ├── PaymentPage.vue      # Step 3 (payment.html)
│   │   │   └── CompletePage.vue     # Step 4 (complete.html)
│   │   ├── components/              # Reusable UI components
│   │   ├── composables/             # Shared logic (useApi, useFormValidation)
│   │   ├── types/                   # TypeScript type definitions
│   │   └── assets/                  # Scoped CSS, images
│   └── public/                      # Static assets (logos, images)
├── caddy/
│   └── Caddyfile                    # Caddy configuration
└── sql/
    └── schema.sql                   # Reference schema (also in migrations/)
```

---

## Phase 0: Prerequisites & Environment

### Step 0.1 — Nix Flake for Dev Shell
- Write `flake.nix` providing:
  - `go` (1.22+)
  - `nodejs` (20+) + `npm`
  - `caddy`
  - `postgresql` (16)
  - `go-jet` (code generator)
- Verify: `nix develop` drops into a shell where all tools are available.
- **Test:** `go version`, `node -v`, `caddy version`, `psql --version`, `jet -h`.

### Step 0.2 — Git Ignore & Initial Commit
- Add `.gitignore` covering: `node_modules/`, `dist/`, `*.exe`, `.env`, `result/` (Nix).
- **Test:** no generated files committed.

---

## Phase 1: Database Schema & Migrations

### Step 1.1 — Design Schema
Based on the mockups, the following entities emerge:

| Table | Key Columns | Notes |
|-------|-------------|-------|
| `beneficiaries` | id, name, description, team_color, avatar_url | Pre-seeded + user-created |
| `registrations` | id, beneficiary_id (FK), full_name, email, phone, can_receive_texts, emergency_name, emergency_phone, emergency_relationship, tshirt_size, waiver_requested, waiver_reason, payment_status, created_at | Main registration record |
| `child_registrations` | id, registration_id (FK), name, tshirt_size | Children of a registration |
| `payments` | id, registration_id (FK), amount_base, amount_processing, amount_donation, total_amount, stripe_payment_intent_id, status, created_at | Payment record |

### Step 1.2 — Write Initial Migration
- Create `backend/migrations/001_initial_schema.up.sql` with `CREATE TABLE` statements.
- Create corresponding `.down.sql` for rollback.
- **Test:** Manually apply migration against a local Postgres; verify tables exist.

### Step 1.3 — Seed Data
- Create `backend/migrations/002_seed_beneficiaries.up.sql` with the 6 mock beneficiaries (Sarah Johnson, Michael Chen, Emma Rodriguez, David Thompson, Lisa Patel, James Wilson).
- **Test:** `SELECT * FROM beneficiaries` returns 6 rows with correct colors and descriptions.

---

## Phase 2: Go Backend — Scaffold

### Step 2.1 — Initialize Go Module
```bash
cd backend && go mod init register.anythingforafriend.com/backend
```

### Step 2.2 — Configuration
- Create `internal/config/config.go`:
  - Reads from environment variables: `DATABASE_URL`, `LISTEN_ADDR` (default `:8080`), `CORS_ORIGIN`, `STRIPE_SECRET_KEY` (placeholder).
  - Uses `os.Getenv` with defaults.
- **Test:** `go build ./internal/config/...` compiles.

### Step 2.3 — Database Connection Pool (pgx)
- Create `internal/store/pool.go`:
  - `func NewPool(ctx context.Context, connStr string) (*pgxpool.Pool, error)`
  - Returns a pgxpool configured with sensible defaults.
- **Test:** Write a small test that connects to a running Postgres and runs `SELECT 1`.

### Step 2.4 — go-jet Code Generation
- Install go-jet.
- Configure jet to point at the Postgres database and output generated types to `internal/model/`.
- Generate types from the initial schema.
- **Test:** `go build ./...` compiles; generated model types can be imported.

### Step 2.5 — Health Check Endpoint
- Create `internal/handler/health.go` — `GET /api/health` returns `{"status": "ok"}` and also pings the database.
- **Test:** `curl http://localhost:8080/api/health` returns 200 and JSON.

### Step 2.6 — HTTP Server Skeleton
- Create `internal/server/server.go`:
  - Wires up `net/http` mux (or `chi`/`gin` if preferred) with:
    - `GET /api/health`
    - CORS middleware (allow Vue dev server origin during development)
    - JSON content-type middleware
    - Timeout middleware
- Create `cmd/server/main.go`: loads config, creates pool, starts server.
- **Smoke test:** Server starts, health endpoint returns 200.

---

## Phase 3: Go Backend — API Endpoints

### Step 3.1 — Beneficiary List
- `GET /api/beneficiaries` — returns all beneficiaries.
- Uses go-jet to query the `beneficiaries` table.
- **Test:** `curl /api/beneficiaries | jq` returns array of 6.

### Step 3.2 — Beneficiary Create
- `POST /api/beneficiaries` — accepts name, description, team_color, team_lead_name, team_lead_phone, team_lead_email.
- Validates required fields; inserts via go-jet.
- **Test:** `curl -X POST ...` creates a new beneficiary; verify in DB.

### Step 3.3 — Registration Create
- `POST /api/registrations` — main registration payload.
- Payload matches register.html form data.
- Validates: required fields, email format, phone format, tshirt size enum.
- Creates registration + child_registrations in a transaction.
- **Test:** `curl -X POST ...` with full payload; verify both tables.

### Step 3.4 — Payment Create (Stripe Integration — Placeholder)
- `POST /api/payments` — expects registration_id from prior step.
- Placeholder: stores payment intent as "pending" without real Stripe call yet.
- Full Stripe integration will be a follow-up task.
- **Test:** Payment record appears in DB with status `pending`.

### Step 3.5 — Registration Confirmation
- `GET /api/registrations/:id` — returns full registration details (for the complete.html page).
- Joins registration + beneficiary + children.
- **Test:** Returns expected JSON structure.

### Step 3.6 — API Integration Tests
- Write Go integration tests that spin up a test database, run migrations, and exercise each endpoint.
- Use `testing` + `httptest` packages.
- **Run:** `go test ./...`

---

## Phase 4: Vue 3 Frontend — Scaffold

### Step 4.1 — Vite Project Initialization
```bash
cd frontend
npm create vite@latest . -- --template vue-ts
npm install
```

### Step 4.2 — Dependencies
```bash
npm install vue-router@4 pinia axios bootstrap bootstrap-icons maplibre-gl
npm install -D @types/node
```

### Step 4.3 — TypeScript Configuration
- Configure `tsconfig.json` for strict mode, path aliases (`@/` → `src/`).
- **Test:** `npx vue-tsc --noEmit` passes.

### Step 4.4 — Vue Router
- Create `src/router/index.ts`:
  - `/` → `HomePage.vue`
  - `/register/beneficiary` → `BeneficiaryPage.vue`
  - `/register/info` → `RegisterPage.vue`
  - `/register/payment` → `PaymentPage.vue`
  - `/register/complete/:id` → `CompletePage.vue`
- **Test:** Navigating between routes renders correct components.

### Step 4.5 — Pinia Store for Registration Flow
- Create `src/stores/registration.ts`:
  - Holds the multi-step form state (selected beneficiary, registrant info, children, payment info).
  - Actions: `setBeneficiary`, `setRegistrantInfo`, `setPaymentInfo`, `submit`.
  - Each step validates its data before allowing navigation to the next step.
- **Test:** Store unit tests (Vitest) verify state transitions.

### Step 4.6 — API Client Layer
- Create `src/composables/useApi.ts` or `src/api/client.ts`:
  - Axios instance pointing at `/api` (same origin in production, proxied in dev).
  - Typed functions: `getBeneficiaries()`, `createRegistration(data)`, `createPayment(data)`, `getRegistration(id)`.
- **Test:** Mock axios with `vitest` to verify correct URLs and payloads.

---

## Phase 5: Vue 3 Frontend — Component Implementation

### Step 5.1 — Extract Common Shell
- `App.vue`: Nav bar (logo, brand), `<router-view />`, footer.
- Extract the color scheme (`--primary-color: #FFD1ED`, `--secondary-color: #580C28`) into a global SCSS/CSS file.

### Step 5.2 — HomePage.vue (Landing)
- Port content from `index.html`.
- Hero section, 4-step cards, CTA button → links to `/register/beneficiary`.
- **Test:** Page renders; CTA navigates to beneficiary page.

### Step 5.3 — BeneficiaryPage.vue (Step 1)
- Port from `beneficiary.html`.
- Loads beneficiaries from API (use Pinia store).
- Existing team cards + "Add New Beneficiary" form.
- Validates selection before enabling "Continue" button.
- **Test:** Team cards render from API data; selecting a team enables the button; creating a new beneficiary posts to API.

### Step 5.4 — RegisterPage.vue (Step 2)
- Port from `register.html`.
- Full contact info form with real-time validation.
- Emergency contact section (optional).
- T-shirt size selector.
- Child registration add/remove (dynamic form array).
- Payment waiver request toggle.
- **Test:** Form validation works; "Continue" enables only when required fields valid; child sections add/remove correctly.

### Step 5.5 — PaymentPage.vue (Step 3)
- Port from `payment.html`.
- Order summary (reads from store: $35 base, $2.50 processing, optional donation).
- Donation buttons and custom amount input update total.
- Card info form (placeholder — will be replaced by Stripe Elements in final).
- **Note:** The mockup itself says "This is a design mockup. The final payment page will be provided by our secure payment vendor." We'll implement a placeholder form that collects card info (NOT stored server-side) and calls the payment API.
- **Test:** Donation buttons update total; form validation enables submit button; submitting calls API and redirects to complete page.

### Step 5.6 — CompletePage.vue (Step 4)
- Port from `complete.html`.
- Loads registration by ID from API.
- Displays confirmation, race details, map (MapLibre GL), calendar add, social sharing.
- **Test:** Page loads and renders all sections; map initializes without errors.

---

## Phase 6: Caddy Configuration

### Step 6.1 — Caddyfile
Create `caddy/Caddyfile`:
```
register.anythingforafriend.com {
    # Development: use self-signed cert
    # Production: Caddy auto-obtains LetsEncrypt
    tls internal

    # Reverse proxy API requests to Go backend
    handle /api/* {
        reverse_proxy localhost:8080
    }

    # Serve Vue static assets
    handle {
        root * /path/to/frontend/dist
        try_files {path} /index.html
        file_server
    }

    # Gzip and security headers
    encode gzip zstd
    header {
        X-Content-Type-Options "nosniff"
        X-Frame-Options "DENY"
        -Server
    }
}
```

### Step 6.2 — Integration Test
- Build frontend: `cd frontend && npm run build` → produces `dist/`.
- Start backend: `go run ./cmd/server`.
- Start Caddy: `caddy run --config caddy/Caddyfile`.
- **Test:** `curl -k https://localhost/api/health` → 200; `curl -k https://localhost/` → returns `index.html`; browser at `https://localhost` loads full Vue app and can call API.

---

## Phase 7: NixOS Deployment

### Step 7.1 — NixOS Module for the Service
- Write a NixOS module that:
  - Creates a systemd service for the Go backend.
  - Configures PostgreSQL with the application database and user.
  - Runs migrations on startup (or via a oneshot service).
  - Configures Caddy with the final domain and LetsEncrypt.
  - Builds and serves the Vue frontend (or serves pre-built dist).

### Step 7.2 — `flake.nix` with `nixosConfigurations`
- Add a `nixosConfigurations` output for the target machine.
- **Test:** `nixos-rebuild dry-activate` on target host succeeds.

---

## Testing Strategy (Checkpoints Along the Way)

| Phase | Test | What It Proves |
|-------|------|----------------|
| 0.1 | `nix develop` — all tools present | Dev environment works |
| 1.2 | `psql` — tables exist after migration | Database schema correct |
| 1.3 | `SELECT * FROM beneficiaries` → 6 rows | Seed data loaded |
| 2.2-2.3 | `go build ./...` compiles | Go toolchain works |
| 2.4 | jet generation produces model types | go-jet connected to DB |
| 2.5 | `curl /api/health` → 200 | Server runs, DB connected |
| 3.1 | `curl /api/beneficiaries` → JSON array | First API endpoint working |
| 3.3 | `curl -X POST /api/registrations` → 201 | Full registration write works |
| 3.6 | `go test ./...` passes | API integration tests green |
| 4.4 | `npm run dev` — Vue app loads | Frontend toolchain works |
| 4.5 | `npx vitest run` — store tests pass | State management works |
| 5.6 | Full flow in browser: beneficiary→info→payment→complete | End-to-end local |
| 6.2 | `curl -k https://localhost/` serves Vue app via Caddy | Caddy reverse proxy works |
| 7.2 | `nixos-rebuild dry-activate` on target succeeds | Deployable on NixOS |

---

## Implementation Order (Recommended)

1. **Phase 0** — Dev environment (Nix flake)
2. **Phase 1** — Database schema & migrations (source of truth for both backend and frontend types)
3. **Phase 2** — Go backend scaffold (config, pool, health endpoint, go-jet generation)
4. **Phase 3** — API endpoints (beneficiaries → registrations → payments → confirmation)
5. **Phase 4** — Frontend scaffold (Vite, Vue Router, Pinia store, API client)
6. **Phase 5** — Frontend components (one page at a time, starting with HomePage)
7. **Phase 6** — Caddy integration (bring everything together behind one origin)
8. **Phase 7** — NixOS deployment

---

## Open Decisions / Deferred Items

1. **Stripe Integration** — The payment mockup is explicitly a placeholder. Real Stripe Elements + webhook handling will replace the card form in Phase 5.5. We'll need Stripe SDK (`github.com/stripe/stripe-go`) on the backend and `@stripe/stripe-js` on the frontend.

2. **Authentication** — No admin/auth in the mockups. If an admin dashboard is needed later (to view registrations, manage beneficiaries), we'll add JWT or session-based auth.

3. **Email Notifications** — The complete.html page references confirmation emails. We'll need email sending (SMTP or SendGrid) as a follow-up.

4. **Race Date** — The mockup shows "June 15, 2024" but the copyright says 2026. The actual race date should be configurable.

5. **Image Hosting** — Mockup uses Unsplash URLs for beneficiary avatars. We should host these locally or use a controlled CDN for production.
