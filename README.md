# theWorks — Hacker101 Challenge Workspace

Practice playground for red-team skill building against [HackerOne 101](https://hacker101.com) practice labs and `*.h1x.com` CTF targets.

## Rules of engagement

- **Only** attack hosts explicitly provisioned by Hacker101 for the current challenge (`/challenge?...` sessions and their assigned `h1x` targets).
- Never scan, fuzz, or replay against out-of-scope hosts, even adjacent ones.
- These are training labs — findings here are practice, not reportable vulnerabilities.

## Layout

```
challenges/
  <NNN>-<topic>/        # one dir per challenge, e.g. 001-xss-reflected
    notes.md            # recon -> hypothesis -> exploitation -> writeup
    exploit/            # scripts, payloads, request files
    artifacts/          # screenshots, responses, extracted flags
_template/              # copy this to start a new challenge
```

Topic buckets used by Hacker101: `xss`, `csrf`, `sqli`, `ssrf`, `ssti`, `idor`, `open-redirect`, `auth`, `misc`.

## Workflow per challenge

1. `cp -r _template challenges/NNN-topic`
2. Fill `notes.md` as you go: scope, recon, hypothesis, PoC, root cause, fix.
3. Keep raw evidence in `artifacts/`; reusable code in `exploit/`.
