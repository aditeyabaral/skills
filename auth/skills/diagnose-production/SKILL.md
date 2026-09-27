---
name: diagnose-production
description: Investigate pesu-auth problems in production or staging - errors, latency, wrong data, outages - using status pages, the deployed version, /metrics accounting and upstream counters, and reproduction, to separate PESU Academy outages and page changes from our bugs and Render issues. Read-only; ends with a recommended fix task. Use when users report failures or monitoring shows trouble.
---

# Diagnosing production and staging

| Environment | URL                                 | Status page                            |
| ----------- | ----------------------------------- | -------------------------------------- |
| Production  | `https://pesuauth.onrender.com`     | `https://xzlk85cp.status.cron-job.org` |
| Staging     | `https://pesuauth-dev.onrender.com` | `https://6ns95sgb.status.cron-job.org` |

The Grafana dashboard linked in the README shows both environments' metrics over time. Both run on
Render's free tier in Singapore: a cold start after idling takes tens of seconds, and restarts reset
all counters.

## 1. Establish facts

```bash
curl -s https://pesuauth.onrender.com/openapi.json | python -c 'import json,sys; print(json.load(sys.stdin)["info"]["version"])'
curl -s -w "\n%{http_code} %{time_total}s\n" https://pesuauth.onrender.com/health
gh run list --repo pesu-dev/auth --workflow deploy-prod.yaml --limit 3      # recent deploys
git log --oneline upstream/main -10                                          # what is in prod
```

## 2. Read the metrics

```bash
curl -s "https://pesuauth.onrender.com/metrics?fmt=json" | python -m json.tool
```

If it answers 401, `METRICS_TOKEN` is set on the service; use a token only if the human provided one
(`-H "Authorization: Bearer <token>"`), and never print it.

Reading guide (definitions in the README "`/metrics`" section):

| Look at                                                              | Meaning                                                                                                                                                                                                    |
| -------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `uptimeSeconds`, `startTimeSeconds`                                  | Recent restart? Counters start from zero at each restart                                                                                                                                                   |
| `failuresByFault.server` vs `.client`                                | Our/upstream faults (5xx) vs caller mistakes (4xx)                                                                                                                                                         |
| `errorsByType`                                                       | Which exception: `AuthenticationError` (wrong passwords, normal), `CSRFTokenError`/`ProfileFetchError` (upstream answered oddly, 502), `ProfileParseError` (page changed, 422), other names (our bug, 500) |
| `profileParseErrors`                                                 | Which part of the profile page broke: `page_structure`, `unknown_field`, `key_missing`, ... → `update-scraper`                                                                                             |
| `upstream.<op>.error`, `responsesByStatus`, `latency.averageSeconds` | PESU down, erroring or slow; compare with our overall `latency`                                                                                                                                            |
| `authenticationResults`                                              | Login success rate; a sudden drop to ~0 with correct users → PESU login changed or down                                                                                                                    |
| `csrfCache.miss`, `csrfRefreshes.failure`, `prefetchTasks.failure`   | Token prefetch failing: upstream issue or a regression in the prefetch code                                                                                                                                |
| `httpClients.created - closed`                                       | Should be about 1 at rest; climbing means a client leak (our bug)                                                                                                                                          |
| `requestsInFlight`                                                   | Stuck requests if it stays high                                                                                                                                                                            |

Check the accounting invariants (`.agents/pesudev-skills/auth/docs/metrics.md`); if they are violated,
the metrics code itself has a bug.

## 3. Reproduce

- Against staging or a local instance (`run-locally`), with the test account, one session.
- Compare with PESU directly: can the test account log in at `https://www.pesuacademy.com/Academy/`
  in a browser (human) or via the app locally?
- Check whether the failing behaviour exists on `upstream/dev` (staging) and on the prod commit.

## 4. Conclude

| Evidence                                                                | Cause              | Next step                                               |
| ----------------------------------------------------------------------- | ------------------ | ------------------------------------------------------- |
| Upstream errors/timeouts, PESU unreachable                              | PESU outage        | Report; nothing to change; monitor                      |
| Parse errors, CSRF 502s, logins failing for valid users, PESU reachable | PESU changed pages | `update-scraper` task                                   |
| Started with a deploy; reproducible locally on that commit              | Our regression     | `fix-bug` task; recommend maintainers consider a revert |
| Slow first request, then fine                                           | Render cold start  | Expected on free tier                                   |
| Client counts climbing, memory growth                                   | Leak               | `change-upstream-client` / `fix-bug`                    |

## Report

Timeline, environment and version, evidence (metric values, status codes), cause with confidence,
recommended next step. Never trigger deploys, rollbacks or config changes, and never include tokens
or personal data.
