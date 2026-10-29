---
name: diagnose-production
description: Investigate pesu-auth problems in production or staging - errors, latency, wrong data, outages - using status pages, the deployed version, /metrics accounting and upstream counters, and reproduction, to separate PESU Academy outages and API changes from our bugs and Render issues. Read-only; ends with a recommended fix task. Use when users report failures or monitoring shows trouble.
---

# Diagnosing production and staging

| Environment | URL                                 | Status page                            |
| ----------- | ----------------------------------- | -------------------------------------- |
| Production  | `https://pesuauth.onrender.com`     | `https://xzlk85cp.status.cron-job.org` |
| Staging     | `https://pesuauth-dev.onrender.com` | `https://6ns95sgb.status.cron-job.org` |

The Grafana dashboard linked from the README and the repository's `docs/metrics.md` shows both
environments' metrics over time. Both run on Render's free tier in Singapore: a cold start after
idling takes tens of seconds, and restarts reset all counters.

## 1. Establish facts

```bash
curl -s -w "\n%{http_code} %{time_total}s\n" https://pesuauth.onrender.com/health      # version and environment
curl -s -w "\n%{http_code} %{time_total}s\n" https://pesuauth-dev.onrender.com/health
gh run list --repo pesu-dev/auth --workflow deploy_prod.yml --limit 3      # recent releases
gh run list --repo pesu-dev/auth --workflow deploy_staging.yml --limit 3   # recent staging deploys
git log --oneline upstream/main -10                                         # what is in production
```

`main` is moved to a commit only after it is released, so `upstream/main` is what production runs
unless a release failed after its deploy (check the latest Deploy to Production run).

`/health` reports the running `version` and its `environment`: `production` on pesuauth and `staging`
on pesuauth-dev. `development` there means `PESU_AUTH_ENVIRONMENT` is not set on that Render
service.

## 2. Read the metrics

```bash
curl -s "https://pesuauth.onrender.com/metrics?fmt=json" | python -m json.tool
```

If it answers 401, `METRICS_TOKEN` is set on the service; use a token only if the human provided one
(`-H "Authorization: Bearer <token>"`), and never print it.

Reading guide (every family is defined in the repository's `docs/metrics.md`):

| Look at                                                              | Meaning                                                                                                                                                                                                                                                                                                        |
| -------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `uptimeSeconds`, `startTimeSeconds`                                  | Recent restart? Counters start from zero at each restart                                                                                                                                                                                                                                                       |
| `failuresByFault.server` vs `.client`                                | Our or upstream faults (5xx) vs caller mistakes (4xx)                                                                                                                                                                                                                                                          |
| `errorsByType`                                                       | Which exception: `AuthenticationError` (wrong passwords, normal), `UpstreamError` (the login failed upstream, 502), `ProfileFetchError` (the profile failed upstream, 502), `ProfileParseError` (PESU's API changed, 422), `RouteNotFoundError`/`MethodNotAllowedError` (scanners), other names (our bug, 500) |
| `profileParseErrors`                                                 | `response_structure`, `missing_field`, `unexpected_value`: PESU's API changed → `update-scraper`. `unknown_campus_code`: a new campus or campus name                                                                                                                                                           |
| `upstream.<op>.error`, `responsesByStatus`, `latency.averageSeconds` | PESU down, erroring or slow; compare with our overall `latency`                                                                                                                                                                                                                                                |
| `authenticationResults`                                              | Login success rate; a sudden drop to about 0 with correct users → PESU's login changed or is down                                                                                                                                                                                                              |
| `httpClients.created - closed`                                       | Should be 0 at rest; climbing means a client leak (our bug)                                                                                                                                                                                                                                                    |
| `requestsInFlight`                                                   | Stuck requests if it stays high                                                                                                                                                                                                                                                                                |

Check the accounting invariants (`.agents/pesudev-skills/auth/docs/metrics.md`); if they are
violated, the metrics code itself has a bug.

## 3. Reproduce

- Against staging or a local instance (`run-locally`), with the test account, a handful of requests.
- Check whether PESU itself works: can the test account sign in to PESU Academy (a human, in the
  app or the browser), or through a local instance?
- Check whether the failing behaviour exists on `upstream/dev` (staging) and on the production commit.

## 4. Conclude

| Evidence                                                                                              | Cause                | Next step                                                 |
| ----------------------------------------------------------------------------------------------------- | -------------------- | --------------------------------------------------------- |
| Upstream errors or timeouts, PESU unreachable                                                         | PESU outage          | Report; nothing to change; monitor                        |
| Parse errors, `missing_field`/`unexpected_value` counts, 502s or 401s for valid users, PESU reachable | PESU changed its API | `update-scraper` task                                     |
| Started with a deploy; reproducible locally on that commit                                            | Our regression       | `fix-bug` task; recommend maintainers consider a revert   |
| Slow first request, then fine                                                                         | Render cold start    | Expected on free tier                                     |
| A deploy fails with `PESU_AUTH_ENVIRONMENT must be ...`, or `/health` reports the wrong `environment` | Render setting       | A maintainer sets `PESU_AUTH_ENVIRONMENT` on that service |
| Client counts climbing, memory growth                                                                 | Leak                 | `change-upstream-client` / `fix-bug`                      |

## Report

Timeline, environment and version, evidence (metric values, status codes), cause with confidence,
recommended next step. Never trigger deploys, rollbacks or config changes, and never include tokens
or personal data.
