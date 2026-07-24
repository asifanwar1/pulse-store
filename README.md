# Pulse Store

Pulse Store is a full e-commerce platform made up of four independent projects that share one FastAPI backend:

| Project | Folder | What it is | Stack |
|---|---|---|---|
| Backend API | [`pulse-store-backend/`](pulse-store-backend) | REST API, database, auth, payments, and the AI agents | FastAPI, SQLAlchemy, PostgreSQL, Alembic |
| Admin Dashboard | [`pulse-store-admin/`](pulse-store-admin) | Internal dashboard for managing products, orders, offers, and the AI agents | React 19, TypeScript, Vite, TanStack Query |
| Customer Web App | [`pulse-store-web/`](pulse-store-web) | Customer-facing storefront (browse, cart, checkout, order tracking) | React 19, TypeScript, Vite, TanStack Query |
| Mobile App | [`pulse-store-app/`](pulse-store-app) | Customer-facing storefront for iOS/Android | Expo (React Native), Expo Router |

Each sub-project has its own README with day-to-day details; this file covers the system as a whole, how the pieces fit together, and how to get everything running locally.

## Contents

- [Architecture](#architecture)
- [AI Agents](#ai-agents)
- [Prerequisites](#prerequisites)
- [Setup & Run](#setup--run)
  - [1. Backend API](#1-backend-api)
  - [2. Admin Dashboard](#2-admin-dashboard)
  - [3. Customer Web App](#3-customer-web-app)
  - [4. Mobile App](#4-mobile-app)
- [Environment Variables](#environment-variables)
- [License](#license)

## Architecture

```
                         ┌─────────────────────┐
                         │  pulse-store-backend │  FastAPI · PostgreSQL · Alembic
                         │   /api/v1/*           │  Auth, Products, Orders, Cart,
                         │                       │  Wallet, Reviews, AI Agents, ...
                         └───────────┬───────────┘
                                     │ REST + SSE
              ┌──────────────────────┼──────────────────────┐
              │                      │                      │
   ┌──────────▼──────────┐ ┌─────────▼─────────┐ ┌──────────▼──────────┐
   │ pulse-store-admin    │ │ pulse-store-web    │ │ pulse-store-app     │
   │ Admin dashboard (Vite)│ │ Customer web (Vite)│ │ Mobile app (Expo)    │
   └───────────────────────┘ └─────────────────────┘ └──────────────────────┘
```

All three clients talk to the same backend over HTTP(S). Only the backend touches the database, Supabase storage, Stripe, and Firebase.

## AI Agents

The backend ships three AI assistants under `/api/v1/ai-agents`, built with [Pydantic AI](https://ai.pydantic.dev/) and configurable per-agent from the admin dashboard (model, system prompt, enabled/disabled):

| Agent | `agent_key` | Purpose | Built in |
|---|---|---|---|
| Product Listing | `product_listing` | Admin describes a product in chat; the agent drafts the listing for the admin to review and confirm before it's created | Admin Dashboard |
| Order Tracking | `order_tracking` | Customer asks about an order; the agent looks it up and answers | Web App, Mobile App |
| Customer Query | `customer_query` | General support chat; escalates to a human support ticket when it can't resolve something | Web App, Mobile App |

**How it works:**
- Agent behavior (model, system prompt, on/off) is stored in the database and editable by an admin — no redeploy needed to change a prompt or swap models.
- Chat runs over **Server-Sent Events** (`POST /api/v1/ai-agents/{agent_key}/chat`), streaming the reply token-by-token and returning a `conversation_id` to continue the thread.
- Unresolved Customer Query conversations become **support tickets**, visible and resolvable from the Admin Dashboard.
- Model selection is pluggable across providers — see `pulse-store-backend/app/features/ai_agents/model_catalog.py` for the current list (Groq, OpenAI, Anthropic). Each provider only becomes available once its API key is set in the backend `.env`.

**To use the AI agents you need at least one provider key** (Groq is the simplest to get started with — free tier, no credit card):
```
GROQ_API_KEY=your-groq-api-key
# and/or
OPENAI_API_KEY=your-openai-api-key
ANTHROPIC_API_KEY=your-anthropic-api-key
```
Without a key configured for the model an agent is set to use, chat requests will stream back an error event instead of a reply — useful for testing the plumbing, but no substitute for setting a real key before a demo.

See [`pulse-store-admin/AI_AGENTS_RELEASE_NOTES.md`](pulse-store-admin/AI_AGENTS_RELEASE_NOTES.md) for the full endpoint reference and integration notes (also relevant to the web and mobile teams building the customer-facing chat).

## Prerequisites

Install these before setting up any of the projects:

- **Python** 3.12+ and `pip`
- **Node.js** 22.17.1+ and `npm`
- **PostgreSQL** 14+ (running locally, or a hosted instance)
- **Git**

Optional, only if you use the related features:
- A [Supabase](https://supabase.com) project (media/file storage)
- A [Stripe](https://dashboard.stripe.com) account in test mode (payments)
- A [Firebase](https://console.firebase.google.com) project (push notifications)
- An API key from [Groq](https://console.groq.com), [OpenAI](https://platform.openai.com), and/or [Anthropic](https://console.anthropic.com) (AI agents)
- [Expo Go](https://expo.dev/go) app or an Android/iOS emulator (mobile app)

## Setup & Run

Clone the repo, then set up each project you need. The backend must be running before any client will work end-to-end.

### 1. Backend API

```bash
cd pulse-store-backend
python -m venv venv
venv\Scripts\activate          # Windows
# source venv/bin/activate     # macOS/Linux

pip install -r requirements.txt
cp .env.example .env           # then fill in the values — see below
```

Create the database, run migrations, then start the server:

```bash
# Make sure DATABASE_URL in .env points at a database that already exists
alembic upgrade head
python scripts/create_admin.py   # creates the first admin user, interactive prompt
uvicorn app.main:app --reload
```

The API is now at `http://localhost:8000`, interactive docs at `http://localhost:8000/docs`.

Run the test suite:
```bash
pytest
```

### 2. Admin Dashboard

```bash
cd pulse-store-admin
npm install
npm run dev
```

Runs at `http://localhost:5173` (Vite's default) and expects the backend at the URL configured in `src/Config.ts` / its `.env`.

### 3. Customer Web App

```bash
cd pulse-store-web
npm install
npm run dev
```

Same Vite setup as the admin dashboard, on its own port.

### 4. Mobile App

```bash
cd pulse-store-app
npm install
cp .env.example .env    # set EXPO_PUBLIC_API_BASE_URL — see the file for the emulator-specific values
npx expo start
```

From the Expo CLI output, choose to open the app in an Android emulator, iOS simulator, Expo Go, or a development build. Note the backend base URL differs by target:

| Target | `EXPO_PUBLIC_API_BASE_URL` |
|---|---|
| Android emulator | `http://10.0.2.2:8000` |
| iOS simulator | `http://localhost:8000` |
| Physical device | `http://<your-machine-LAN-IP>:8000` |

## Environment Variables

Each project keeps its own `.env` (never committed — see `.gitignore`), copied from a `.env.example` in the same folder:

- `pulse-store-backend/.env.example` — database URL, JWT secret, CORS origins, Supabase, Stripe, and AI provider keys
- `pulse-store-app/.env.example` — backend base URL for the mobile app
- `pulse-store-admin` / `pulse-store-web` — configured via `src/Config.ts`; check that file for the variables each app reads

## License

This project is licensed under the MIT License — see [LICENSE](LICENSE) for the full text.
