# 004 — Encrypted Pastebin: AES-CBC undone with its own error pages

- **Target:** `<instance>.ctf.hacker101.com` (Hacker101 CTF lab) — "military-grade 128-bit AES; the key never touches our database"
- **Status:** 3/4 flags captured before the per-session instance expired mid-exploit

## App model

Stateless pastebin: `POST /` (title, body) → redirect to `/?post=<blob>` where blob is a URL-mangled base64 (`~!-` → `=+/`) of `IV ‖ AES-128-CBC(json)`. The link **is** the storage — the page renders straight from decrypted JSON:

```json
{"flag": "^FLAG^…$FLAG$", "id": "N", "key": "…"}
```

Title/body go to a DB; the link carries a per-instance flag, the row id, and a random key. Hosted instance ran with debugging on: malformed links return **full Python tracebacks** (framework, file names, library versions — a fingerprinting gift).

## Flag 1 — free on the error page

Break the base64 (`?post=AAAA`), or send wrong length / 16-byte-only blobs. Every error page embeds flag #1 plus stack frames that confirm: py2.7 Flask, `AES.new(staticKey, AES.MODE_CBC, iv)`, custom `unpad` raising a distinctive `PaddingException`.

## Flag 2 — padding-oracle decryption of your own link

The oracle: `PaddingException` ⇔ invalid; **any other response** (JSON error, UnicodeDecodeError — even success) ⇔ valid padding. Decrypt block-by-block with blobs truncated at the target (`predecessor‖C_i` — padding is only ever checked on the final block), scanning one byte per position right-to-left. Blocks 0–5 hold the embedded flag.

Two traps cost real time:
1. Probes that don't truncate answer about the untouched tail — "True for everything".
2. Their `unpad` accepts padding=0, making the first search position ambiguous → collect all candidates and backtrack.

## Flag 3 — forging a link without the key

A padding oracle encrypts too: solve `D(R)` for a random block via the same probing, then set the predecessor to `D(R) ⊕ target`. One sequential solve per block (the predecessor slot is itself ciphertext needing its own solve). Single-block target `{"id": "1"}` → server dutifully fetches post #1 — admin's — and prints *"Attempting to decrypt page with title: <flag #3>"*. The id field is later string-interpolated into SQL (`'SELECT … WHERE id=%s' % post['id']`), proven by quote-breaking it.

## Flag 4 (unfinished) — exfiltrating the bot's link via UNION

The tracking table logs request headers of every visit — including an admin-bot whose Referer contains a secret post link. Planned: forge `{"id": "0 UNION SELECT GROUP_CONCAT(headers),1 FROM tracking-- -"}` (5 chained solves) and read the dump off the same leak channel. Instance expired at 3/5 chain links; solver + assembly are scripted for a fresh instance (~1–1.5 h).

## Lessons

- Unauthenticated CBC + any padding signal = full plaintext recovery **and** forgery. Encrypt-then-MAC, uniform error pages, and debug off — the app failed all three.
- Traceback leakage alone mapped the entire vulnerability surface before exploit #1 fired.
- Pacing matters against shared infra: >2–3 probes/s turns the LB into an oracle-poisoning machine (502s with `<h1>` masquerading as renders).
