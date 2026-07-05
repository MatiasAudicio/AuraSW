# Security engineering notes

This project handles identity documents, biometric verification data, and
private messaging for real users, so security work wasn't an afterthought —
it was ongoing, iterative hardening across the whole backend. Some concrete
examples of what that looked like in practice.

## Database access model

The backend is the single source of truth for data access: it talks to
Postgres through the Supabase service-role key, and the frontend never
touches the Supabase client directly (no `createClient`, no
`supabase.from(...)`, no public Supabase env vars in the frontend bundle).
Every read/write goes through FastAPI, where authorization is enforced.

That access model mattered for a real finding: **25 tables had Row Level
Security disabled**, which meant anyone with the public `anon` key could
read or write them directly via Supabase's auto-generated REST API
(PostgREST), completely bypassing the FastAPI authorization layer.

Because the app itself never used that direct path, enabling RLS with a
deny-all policy for `anon`/`authenticated` closed the hole with zero risk of
breaking existing functionality — the backend keeps working via the
service-role key, which RLS does not restrict. Rather than flipping RLS on
for all 25 tables at once, the rollout was staged by blast radius:

| Batch | Risk | Examples |
|---|---|---|
| 1 | Low / ephemeral | typing indicators, push subscription tokens, conversation settings |
| 2 | Medium | comments, saves, story reactions/highlights, poll votes, events |
| 3 | High | follows, blocks, reports, matches, private group messages, paid album content |

Each batch was applied as its own migration and smoke-tested against the
real user flows it touched before moving to the next one. `backend/supabase/`
contains the actual migration files.

## Other hardening done along the way

A non-exhaustive list of vulnerability classes found and fixed during
security review passes on this codebase:

- **Timing attacks** on auth-adjacent comparisons (login, token validation)
- **XSS** in notification content (unsanitized user input rendered as HTML)
- **TOCTOU races** — token refresh, reacting/saving on posts that get
  deleted mid-request, concurrent profile edits
- **Signed URL path traversal** — media URLs validated against directory
  escape before signing
- **Open redirects** in auth/deep-link flows
- **bcrypt-based DoS** — unbounded login attempts hashing on every request
- **Decompression bombs** on user-uploaded media
- **Visibility/ownership bypass** — endpoints that returned or mutated rows
  without checking the requester actually owned or could see them
- **Rate limiting** on expensive or spammable endpoints (heartbeat, profile
  views, saves, follow notifications)

## Identity verification (KYC)

Every account is verified with government ID + biometric matching
(MetaMap) before it can access the platform, with a simulation mode for
local development and a manual-review fallback (admin-generated one-time
keys → manual approval queue) for cases the automated check can't resolve.

## Auth & sessions

JWT access/refresh tokens with rotation, optional TOTP-based 2FA, and a
sessions panel where users can see and revoke active devices individually.
