# 001 — Level 0: hidden data in a fake image

- **Target:** `<instance>.ctf.hacker101.com` (per-session Hacker101 CTF lab)
- **Objective:** find the flag
- **Status:** solved

## Recon

- `/` renders "Welcome to level 0." with one referenced asset: CSS `background-image: url("background.png")`.
- `robots.txt` and common paths (`/flag`, `/admin`, `/.git/HEAD`, …) all return Werkzeug-style 404s.
- No `Set-Cookie` on any response; no HTML comments.

## Hypotheses

| # | Idea                                                    | Tested | Result                                    |
|---|---------------------------------------------------------|--------|-------------------------------------------|
| 1 | Flag hidden in the only referenced asset                | yes    | hit — file is not an image at all         |
| 2 | Cookie/header disclosure                                | yes    | dead end                                  |
| 3 | Path brute force                                        | yes    | dead end, uniform 404s                    |

## Exploitation

1. Fetch the asset: `curl -s "$T/background.png" -o bg.png` → only **76 bytes**.
2. Magic bytes are not PNG (`89 50 4e 47` absent) — it is a plain text file wearing an image extension.
3. The entire body is the flag token, wrapped in the Hacker101 CTF format `^FLAG^<64-hex>$FLAG$`. Submit that string on ctf.hacker101.com.

## Root cause

A secret stored in a client-visible static asset, "protected" only by an image filename. Hidden-by-extension is not access control.

## Fix

Never ship secrets to the client; serve files with truthful types; if data must reach the browser, treat it as public.

## Flag / result

`^FLAG^<64-hex — read yours from your own instance's background.png>$FLAG$`

**Lesson:** view source → fetch every referenced asset → inspect raw bytes. Extensions and content-types lie; magic numbers don't.
