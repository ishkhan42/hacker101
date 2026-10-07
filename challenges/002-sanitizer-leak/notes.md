# 002 — Micro-CMS: the XSS sanitizer leaks its own secret

- **Target:** `<instance>.ctf.hacker101.com` (per-session Hacker101 CTF lab)
- **App:** "Micro-CMS" — create/edit pages, markdown rendered server-side, hint: *"Markdown is supported, but scripts are not"*
- **Status:** solved

## Recon

- CRUD at `/page/<id>`; content and title render through a markdown pipeline that passes raw HTML through.
- Werkzeug 404s; ids route-converted to int.

## Hypotheses

| # | Idea                                        | Tested | Result                                  |
|---|---------------------------------------------|--------|-----------------------------------------|
| 1 | SSTI (`{{7*7}}`, `{{config}}`) in title/body, create/edit | yes | dead — always literal; `${}`, ERB, `#{}` too |
| 2 | SQLi on page id                             | yes    | dead — int route converter 404s first   |
| 3 | python-markdown smarty file inclusion `--8<--` (abs + rel) | yes | dead — literal; quote fingerprint shows smarty off |
| 4 | Hidden endpoints (`/admin /flag /api /.env …`) | yes  | dead — uniform 404                      |
| 5 | XSS filter behavior diffing                 | yes    | **HIT — sanitizer leaked the flag**     |

## Exploitation

Submitted as page body:

```html
<script>alert(1)</script>

<img src=x onerror=alert(2)>

[a](javascript:alert(3))
```

Rendered back:

```html
<scrubbed>alert(1)</scrubbed>

<p><img src=x flag="^FLAG^<64-hex>$FLAG$" onerror=alert(2)></p>

<p><a href="javascrubbed:alert(3)">a</a></p>
```

The sanitizer scrubs `<script>` and `javascript:` — but for the event-handler attribute it injected a `flag="…"` attribute containing the challenge flag. The flag constant lives inside the sanitizer's scrub logic, and its replacement output reflected it into the page. Extract: regex `\^FLAG\^[a-f0-9]{64}\$FLAG\$` on the rendered page; verify across two fetches.

## Root cause

A security filter whose internal constants/logics are observable from its output — dangerous-input handling embedded a secret directly in HTML replacements.

## Fix

Scrubbers must remove/neutralize without echoing internal state; keep secrets out of request-handling code paths entirely; deny-list sanitizers are weak — prefer allow-list output encoding (e.g. bleach/CSP).

**Lesson:** feed every filtering path a canary payload and diff input vs output byte-for-byte. Replacements, re-encodings, and injected attributes talk — the filter itself was the "victim" here, no XSS execution needed.
