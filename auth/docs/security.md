# Security

pesu-auth handles university credentials for every caller. A leak here is a leak of students'
PESU accounts. Treat every change to request handling, logging or upstream code as security
relevant.

## What is sensitive

| Data                                   | Where it appears             | Rule                                                                                                                                                                                       |
| -------------------------------------- | ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Passwords                              | request body, the login POST | Never logged, stored, echoed, or put in an exception message or metric                                                                                                                     |
| Usernames (SRN/PRN/email/phone)        | logs (`user=...`)            | Already logged at INFO; do not add new places                                                                                                                                              |
| Profile data (name, email, phone, ...) | response, logs               | Logged at INFO by design, so a login can be traced to what PESU returned: the scraped profile (`app/pesu.py`) and the auth result (`app/app.py`). Not a defect; do not remove or redact it |
| CSRF tokens, session cookies           | upstream client              | Never log new occurrences; never reuse a client across callers                                                                                                                             |
| `METRICS_TOKEN`                        | environment                  | Never logged; compared with `secrets.compare_digest` on bytes                                                                                                                              |
| `TEST_*` credentials                   | `.env`, CI secrets           | Never printed, committed, pasted into PRs, issues or logs, and never given to Copilot's cloud agent (the repo's `copilot` environment or `copilot-setup-steps.yml`)                        |

## Threats and the controls that address them

- **Credential exposure in logs.** The validation handler logs only `type`, `loc`, `msg` of each
  error, never `input` (which is the whole body for a missing field). Any new handler or log line must
  follow the same rule. Test it with `caplog`: assert the password string is absent.
- **Session mixing.** Each login uses its own `httpx2.AsyncClient` taken from the cache under a lock
  and closed afterwards. Never share an authenticated client between requests.
- **Timing attacks on the metrics token.** `secrets.compare_digest` on bytes (header latin-1,
  configured token utf-8 surrogateescape). Do not replace with `==`, and do not compare `str`s
  (non-ASCII input would raise and become a 500).
- **Metric label cardinality (memory DoS).** Labels are clamped to closed sets; see `.agents/pesudev-skills/auth/docs/metrics.md`.
- **Mass assignment / unexpected input.** `RequestModel` is strict with `extra="forbid"`.
- **Information disclosure in errors.** 500s return a generic message; details go to logs only.
  Upstream error text is never forwarded to the caller.
- **Resource exhaustion.** Upstream timeout is 10s; clients are always closed; the CSRF fetch on a
  cold cache happens outside the lock.
- **Supply chain.** Dependencies are pinned in `uv.lock`; the Docker build uses `--frozen`.

## Review checklist for any change

- No new log line, exception message, metric label or response field contains a password, token or
  new personal data.
- No new unbounded label, unbounded cache, or unbounded loop against upstream.
- New inputs are validated by a strict pydantic model; unknown keys rejected.
- New endpoints: is it safe to leave unauthenticated? `/health` must stay open; anything exposing
  operational data should follow the `METRICS_TOKEN` pattern.
- New dependencies: maintained, widely used, license compatible with MIT, pinned via `uv lock`.
- Secrets are read from the environment, never hard-coded, and never from `.env` in app code (the
  app does not read `.env`).

## Auditing dependencies

```bash
uvx --python 3.14 pip-audit --strict --require-hashes --disable-pip -r <(uv export --frozen --no-dev --no-emit-project)
```

Report findings with the advisory ID and the fixed version; upgrading follows the
`update-dependencies` skill.

## Reporting

Vulnerabilities are reported privately to maintainers (`.github/SECURITY.md`), never in a public
issue or PR. An agent that finds a vulnerability stops, writes the finding in its report to the
human, and does not open a public PR describing it.
