# WhatsApp Order Store

> **"Your online store, powered by WhatsApp."**

A mini e-commerce platform for small businesses (fashion, food, beauty, electronics, etc.) that lets
customers browse a storefront, build a cart, and send their order straight to the business owner's
WhatsApp — no payment gateway, no complicated checkout.

## Architecture

```
What'sApp system/
├── backend/     FastAPI + PostgreSQL (async SQLAlchemy 2.0 + Alembic)
└── frontend/    React + TypeScript + Vite + Tailwind CSS
```

### Why FastAPI (Python)

- **Pydantic validation at every boundary** — request/response schemas are declared once and enforce
  themselves; this matters a lot for a platform with dozens of untrusted public endpoints (checkout,
  order creation) that must never trust client-sent prices/stock.
- **Async native** — the storefront is read-heavy (product grids, store pages) and async SQLAlchemy +
  asyncpg keeps request latency low without extra infrastructure.
- **Dependency-injected auth/ownership** — `get_current_user` / `require_store_owner` dependencies make
  "a business owner can only touch their own store" a one-line guarantee on every route instead of
  scattered `if` checks.
- **Free OpenAPI docs** (`/docs`) — useful for a project that will keep growing (Mobile Money, WhatsApp
  Business API, etc. per the roadmap) without hand-written API docs going stale.

### Why React + TypeScript + Vite + Tailwind

- Tailwind lets the brand palette (`#2563EB` primary / `#0F172A` dark / `#F8FAFC` background /
  `#16A34A` success) live as design tokens in one config file, and lets each store owner's
  `primary_color` override the theme at runtime via CSS variables — no per-component theming code.
- TypeScript types are generated to mirror the backend Pydantic schemas so the cart/checkout/WhatsApp
  message pipeline is type-safe end to end.
- Zustand (not Redux) for cart + auth state — minimal boilerplate, persists the cart to `localStorage`
  so a customer who refreshes mid-checkout doesn't lose their cart.

## Database schema (see `backend/app/models/`)

`users` → `stores` (1 owner : 1 store for MVP) → `categories`, `products` → `product_variants`
`stores` → `orders` → `order_items`

Full column-level design is documented at the top of each model file.

## Order flow (see PROJECT spec §19)

1. Customer builds cart client-side (localStorage, no account needed).
2. Checkout form collects name / WhatsApp number / delivery location / note.
3. `POST /api/public/stores/{slug}/orders` — **server** re-validates every product's live price/stock/
   active status (never trusts the client's cached prices), creates the `Order` + `OrderItem` rows,
   generates a human-readable order number (`ORD-YYYYMMDD-XXXX`), and builds the formatted WhatsApp
   message server-side.
4. Response includes a `whatsapp_url` (`https://wa.me/<number>?text=<encoded message>`). The frontend
   opens it in a new tab/window — WhatsApp opens with the message pre-filled, and the customer must
   press **Send** themselves. The order is already recorded in the dashboard regardless of whether the
   customer actually presses Send (status starts as `new`).

## Running locally

See `backend/README.md` and `frontend/README.md` for setup. Quick start:

```bash
# 1. Postgres (or use docker: docker run -p 5432:5432 -e POSTGRES_PASSWORD=postgres postgres:16)
# 2. Backend
cd backend
python -m venv .venv && .venv/Scripts/activate   # Windows
pip install -r requirements.txt
copy .env.example .env                            # fill in DATABASE_URL etc.
alembic upgrade head
python -m app.seed                                 # loads the Nova Fashion demo store
uvicorn app.main:app --reload

# 3. Frontend
cd frontend
npm install
copy .env.example .env
npm run dev
```

## What's implemented vs. what needs connecting

Implemented and functional: auth, store onboarding/settings, category & product CRUD with image
upload, order creation + status management, WhatsApp message/link generation, storefront (home,
product detail, cart, checkout, confirmation), business dashboard, platform admin, QR code + share
links, demo seed data.

Needs a real provider before production:
- **Image storage** — currently saves to `backend/uploads/` and serves as static files. Swap
  `app/services/storage.py` for an S3-compatible client; the interface is already isolated there.
- **Email** — password reset generates a token and logs it (`app/services/email.py` has a single
  `send_email()` stub) instead of sending real email; plug in SES/SendGrid/Postmark there.
- **Payments / Mobile Money / WhatsApp Business API** — intentionally out of scope for MVP; the
  `orders.status` enum and `stores` settings table are structured so these can be added without a
  schema rewrite (see PROJECT spec §27).
