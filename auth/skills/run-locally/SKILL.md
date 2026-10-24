---
name: run-locally
description: Run pesu-auth locally from source or as the Docker image and smoke-test every endpoint (the API Explorer, Swagger, /health, /authenticate, /metrics with and without METRICS_TOKEN, /readme, unknown paths), reproducing what the CI container checks verify. Use to verify a change end to end, reproduce a bug, or check the image builds.
---

# Running pesu-auth locally

## From source

```bash
uv run python -m app.app --debug          # http://localhost:5000, DEBUG logs, auto-reload
uv run python -m app.app --port 5001      # another port (or PORT=5001)
METRICS_TOKEN=local-secret uv run python -m app.app   # with /metrics protected
```

Startup makes no call to PESU Academy, so the app starts offline; only `/authenticate` needs to
reach pesuacademy.com. Run the server in the background or a second shell, and stop it when you are
done.

## Smoke test

```bash
curl -s -o /dev/null -w "%{http_code}\n" localhost:5000/                  # 200 API Explorer
curl -s -o /dev/null -w "%{http_code}\n" localhost:5000/docs              # 200 Swagger UI
curl -s localhost:5000/health                                             # {"status":true,"message":"ok",...}
curl -sI localhost:5000/health                                            # HEAD 200, with the security headers
curl -s -o /dev/null -w "%{http_code} %{redirect_url}\n" localhost:5000/readme   # 308 https://github.com/pesu-dev/auth
curl -s localhost:5000/metrics | head -5                                  # Prometheus text
curl -s "localhost:5000/metrics?fmt=json" | python -m json.tool | head -20
curl -s "localhost:5000/metrics?fmt=xml"                                  # 400, standard error body
curl -s localhost:5000/nope                                               # 404, standard error body
curl -s -X POST localhost:5000/authenticate -H 'Content-Type: application/json' \
  -d '{"username": "", "password": "x"}'                                  # 400 validation body
curl -s -X POST localhost:5000/authenticate -H 'Content-Type: application/json' \
  -d '{"username": "INVALID_USER", "password": "wrong"}'                  # 401 (real PESU call)
```

A real login with the test account (do not print the response's profile values into logs or
reports):

```bash
set -a; . ./.env; set +a
curl -s -X POST localhost:5000/authenticate -H 'Content-Type: application/json' \
  -d "{\"username\": \"$TEST_PRN\", \"password\": \"$TEST_PASSWORD\", \"profile\": true}" \
  | python -c 'import json,sys; b=json.load(sys.stdin); p=b.get("profile", {}); print(b["status"], sorted(k for k, v in p.items() if v is None))'
```

That prints the status and which fields came back null, never the values.

With `METRICS_TOKEN` set: `/metrics` without a header is 401 with `WWW-Authenticate: Bearer`
(`curl -si`), and `-H "Authorization: Bearer local-secret"` is 200.

## Docker (what the CI container checks verify)

```bash
docker build . --tag pesu-auth-test
docker run --rm pesu-auth-test id -u                                     # 10001
docker run --rm --name pesu-auth-smoke -d -p 5000:5000 pesu-auth-test
sleep 5 && curl --fail --max-time 10 http://localhost:5000/health && echo OK
docker logs pesu-auth-smoke | tail -20
docker stop pesu-auth-smoke
```

Run it whenever the change touches runtime code paths used at startup, dependencies, `uv.lock` or
the `Dockerfile`. The image copies only `pyproject.toml`, `uv.lock`, `README.md` and `app/` into the
build, and only the virtualenv and `app/` into the runtime image; a new runtime file outside `app/`
must be added to the `Dockerfile`.

## Report

List each request with its status and the relevant part of the body (never credentials or personal
data), and whether Docker was built, ran as UID 10001 and answered.
