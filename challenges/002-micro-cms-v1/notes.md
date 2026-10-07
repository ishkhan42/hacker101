# 002 — Micro-CMS v1: four flags, four input-handling sins

- **Target:** `<instance>.ctf.hacker101.com` (per-session Hacker101 CTF lab)
- **App:** "Micro-CMS" — create/edit pages, markdown rendered server-side, hint: *"Markdown is supported, but scripts are not"*
- **Status:** solved 4/4

## App model

CRUD only: `GET /` (page list), `/page/create`, `/page/<id>` (rendered view), `/page/edit/<id>`. Werkzeug errors, no auth, no cookies — fully stateless. Titles render **raw** in the list; bodies go through an HTML-aware scrubber that passes most raw HTML (`data:`, `file:`, `<style>`, `<meta>`, `<object>` all survive).

## The four flags

| # | Surface | Technique |
|---|---------|-----------|
| 1 | Scrubbed event handler in body | Sanitizer injected a `flag="…"` attribute while processing `on*` — the filter leaked its own embedded constant |
| 2 | Page **title** rendered raw in list view | Dangerous titles trigger an app-injected `<script>alert("<flag>")</script>`; titles render unescaped into `/`, so any XSS attempt pays out without execution |
| 3 | `GET /page/edit/<id>` of a hidden page | The edit route has no access check while the view route 403s on private pages — classic broken access control read via a sibling route |
| 4 | **URL path parameter** SQLi | `GET /page/edit/5'` → SQL syntax error path renders the flag; `' OR '1'='1` tautology confirms injection point |

### Flag 1 — the filter is the victim

Submitted as body: `<script>alert(1)</script>` / `<img src=x onerror=alert(2)>` / `[a](javascript:alert(3))`.
Rendered back:

```html
<scrubbed>alert(1)</scrubbed>
<p><img src=x flag="^FLAG^<64-hex>$FLAG$" onerror=alert(2)></p>
<p><a href="javascrubbed:alert(3)">a</a></p>
```

Rule map (all case-insensitive): `on*` attrs → append `flag="<secret>"`; `<script` → `<scrubbed>`; `javascript:`/`vbscript:` → `…scrubbed:`. Only the handler rule carries a constant. Extract with `\^FLAG\^[a-f0-9]{64}\$FLAG\$` on rendered output.

### Flag 2 — raw titles

Create a page whose title is an HTML fragment; the homepage list renders it unescaped and additionally injects an alert script containing flag #2 next to dangerous entries. One rule, one constant.

### Flag 3 — private page via the window

`/page/<id>` returns 403 for a hidden page (ids visible in the sequence gaps); `GET /page/edit/<same id>` returns 200 with the body in a `<textarea>`. Read-only ACL on one route, none on its twin.

### Flag 4 — quote-test every route, individually

`/page/<id>` uses an int converter (quote → 404), which "proved" SQLi was dead app-wide. It wasn't: **`/page/edit/<id>` interpolates the raw path segment into SQL.** `GET /page/edit/5'` responds 200 with a 76-byte body — exactly flag #4; `/page/edit/5%27%20OR%20%271%27=%271` returns it too. Earlier "flaky" 500s on odd ids (`edit/05`, `edit/5 `) were SQL syntax errors, not routing noise.

## Red herring log (what cost time)

The private-page edit form accepts a `private` field whose hex-decoded value gates saves; one 302 slipped through, then identical requests gave 404/500 — hours went into key-derivation and worker-fleet models. Irrelevant to flag #4: the official per-flag hints (visible on the CTF platform) plus open-source writeup mirrors name each vulnerability class outright. Check a challenge's own metadata before deep black-box modeling.

## Fixes

1. Allow-list output encoding (bleach + CSP), never deny-list scrubbing; keep secrets out of request-handling code paths.
2. Encode titles in every context, not just the "obvious" one.
3. Enforce access checks centrally (decorator/middleware), not per-route by memory.
4. Parameterized queries for **route path parameters** too — they are inputs like any other.

**Lessons:** diff filters byte-for-byte; a 500 on an odd URL is a fingerprint; route converters are per-route, so "SQLi dead here" never generalizes; read the hints.
