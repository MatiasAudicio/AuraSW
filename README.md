# Aura SW

Full-stack social platform with document + biometric identity verification (KYC), built solo end-to-end: backend, frontend, database design, security hardening and deployment.

**What it is:** Aura SW (aurasw.club) is a verified adult lifestyle community for Argentina — think a social network where every member proves their identity (DNI + biometrics) before joining. The identity-verification layer is the core differentiator, and most of the engineering effort went into the security and trust model around it (KYC flow, RLS, rate limiting, session/2FA handling, media protection) rather than the subject matter itself.

## Stack

| Layer | Technology |
|---|---|
| Backend | FastAPI (Python 3.11+) |
| Frontend | React + TypeScript + Vite (PWA) |
| Database | Supabase (PostgreSQL, Row Level Security) |
| Auth | JWT + refresh tokens, TOTP 2FA, session management |
| Media | Supabase Storage, signed URLs, watermarking |
| Payments | Stripe checkout + webhooks |

## Architecture

- **~27 REST routers** under `/api/v1`: auth, KYC, admin, payments, feed, messaging, notifications, profiles, discovery, groups, events, reviews, and more.
- **KYC pipeline**: MetaMap integration (document + biometric verification) with a simulation mode for local dev, master-key-based manual approval flow, and an admin backoffice to review pending users.
- **Security**: Postgres RLS across all user-facing tables, JWT access/refresh rotation, TOTP 2FA, per-endpoint rate limiting, signed media URLs, ownership checks on every mutation, input validation against injection/traversal/DoS-style edge cases.
- **Real-time-ish social features**: feed with posts/stories/polls, nested comments, reactions, follows, DMs with view-once media and typing indicators, push notifications (Web Push/VAPID).
- **Media pipeline**: upload → watermark → Supabase Storage, with signed, time-limited URLs (no public media buckets).

## Local setup

### Requirements
- Node.js 20+
- Python 3.11+
- A Supabase project (free tier is enough for dev)

### 1. Database (Supabase)
1. Create a new Supabase project.
2. Run the SQL files under `supabase/migrations/` in order via the SQL Editor.

### 2. Backend
```bash
cd backend
python -m venv venv
venv\Scripts\activate   # Windows
# source venv/bin/activate   # Mac/Linux

pip install -r requirements.txt
cp .env.example .env
# Fill in Supabase URL/keys, JWT_SECRET_KEY, and set METAMAP_SIMULATION_MODE=true

python main.py
# API on http://localhost:8000  (docs at /docs in dev mode)
```

### 3. Frontend
```bash
cd frontend
npm install
cp .env.example .env   # VITE_API_URL=http://localhost:8000/api/v1
npm run dev
# App on http://localhost:5173
```

### First admin user
```sql
UPDATE users SET role = 'admin', status = 'active' WHERE email = 'your-email@example.com';
```

## Notes

This repo is a code showcase extracted from a private working repository — commit history was reset and deployment secrets/infrastructure IDs were stripped before publishing. It won't run against production infrastructure without your own Supabase/Stripe/MetaMap credentials.
