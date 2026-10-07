# 003 — Micro-CMS v2: auth bolted on, three ways in

- **Target:** `<instance>.ctf.hacker101.com` (Hacker101 CTF lab, per-session)
- **App:** Micro-CMS again — now with login (`l2session` cookie), MariaDB backend behind a py2.7 Flask app; anonymous visitors see only public pages
- **Status:** solved 3/3

## Setup

Homepage lists public pages; everything else redirects to `/login`. A private page exists (id gap + 403 on `/page/<n>`). Login builds SQL by string interpolation of `username` — quote it and the app 500s. Hosted instances run debugger-off, so no tracebacks (writeup screenshots with tracebacks are local runs).

## Flag 1 — UNION login bypass → private page

```
POST /login   username: ' UNION SELECT '123' AS password#   password: 123
```

The forged row satisfies the password comparison; response sets an `admin:true` session cookie. `/page/<private id>` then renders flag #1.

## Flag 2 — auth twin missing on POST

GET `/page/edit/1` → 302 `/login`. **POST to the same URL is unauthenticated** and answers with flag #2 in the response body (any page id, any data). Auth middleware applied per-route-by-memory instead of per-resource: verbs are separate routes.

## Flag 3 — boolean-blind credential extraction

Flag #3 is emitted **only on authentic login** — a forged UNION session doesn't earn it. The login form differentiates `Unknown user` (0 rows) vs `Invalid password` (≥1 row): one bit per request, straight into the WHERE clause.

```
username: x' OR LENGTH((SELECT username FROM admins LIMIT 1))=8#        → Unknown user / Invalid password
username: x' OR ORD(SUBSTRING((SELECT password FROM admins LIMIT 1),3,1))<=77#
```

Length probe per field, then binary-search each character's ordinal (~5 probes/char). Both credentials recovered in ~60 sequential keep-alive requests (<2 min); logging in with them returns flag #3 directly.

Notes: the app escapes `%` → `%%` (kills LIKE wildcard tricks) but not `#` comments or subqueries. Wordlist brute force also works (names are first-name-ish) — blind extraction is just deterministic and fast.

## Lessons

1. Per-route auth checks rot: enumerate **methods per path**, not paths per app.
2. Any two-state response on an injectable parameter = full data-exfiltration channel; binary-search ordinals rather than spraying candidates.
3. When the prize binds to real authentication, bypass ≠ extraction — do both.
4. Read the challenge's official hints + open-source writeups first: all three flag classes were known in minutes (see 002's red-herring log for the counterexample).
