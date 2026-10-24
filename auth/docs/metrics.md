# Metrics

pesu-auth exposes in-process counters at `/metrics`. The repository's `docs/metrics.md` is the full
reference: how collection works, what every family means, the accounting rules, both payload
examples, and protecting and scraping the endpoint. The README section "`/metrics`" covers only the
parameters and responses. This file is the maintainer's view: where each metric is recorded, the
rules they follow, and everything that must change together when one is added.

## Where each metric is recorded

| Layer               | File                        | Records                                                                                                                                                                                                                                               |
| ------------------- | --------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| HTTP middleware     | `app/metrics/middleware.py` | `requests_total`, `requests_success_total`, `requests_failed_total`, `responses_total{status}`, `route_requests_total{method,route}`, `failures_total{fault}`, `request_latency_seconds`, `route_latency_seconds{method,route}`, `requests_in_flight` |
| Exception handlers  | `app/app.py`                | `errors_total{type}`, `validation_errors_total{field}`                                                                                                                                                                                                |
| Routes and lifespan | `app/app.py`                | `authentication_requests_total{profile}`, `authentication_results_total{result}`, `lifespan_events_total{event}`                                                                                                                                      |
| Upstream client     | `app/pesu.py`               | `upstream_*{operation,...}`, `http_clients_total{event}`, `profile_field_filtering_total{enabled}`, `profile_parse_errors_total{reason}`                                                                                                              |
| Collector           | `app/metrics/collector.py`  | `process_start_time_seconds` (seeded at construction)                                                                                                                                                                                                 |

Record a fact in **exactly one** layer and one family. The middleware and the handlers write to
different families on purpose, so one failed request is one status sample and one error sample.

## Rules

1. **Bounded labels.** A label value must come from a closed set the code controls. Never use a raw
   path, a username, a request field value, a value from PESU or an exception message as a label.
   Caller-controlled input is clamped first: `method_label` (unknown verb → `<other>`), `route_label`
   (unmatched path → `<unmatched>`), `KNOWN_REQUEST_FIELDS` (unknown field → `other`). Unbounded
   labels are a memory DoS on a public endpoint.
1. **Declared labels only.** `MetricsCollector._key` raises `ValueError` if the labels passed do not
   exactly match the family's declared `labels` tuple. Tests will catch a mismatch.
1. **Counters only go up**, except the `requests_in_flight` gauge, which is decremented in a
   `finally`.
1. **Monotonic clock** (`time.perf_counter`) for latency; wall clock only for timestamps.
1. **Summaries expose `_sum` and `_count`**, not quantiles.
1. **Unlabelled families are seeded at zero** in `MetricsCollector.__init__`, so they exist from
   the first scrape.
1. **Accounting invariants must keep holding** (they are how an operator decides whether to trust
   the numbers):
   - `requests.success + requests.failed + requestsInFlight == requests.total` (minus abandoned requests)
   - `sum(responsesByStatus) == requests.success + requests.failed`
   - `sum(errorsByType) == requests.failed` once nothing is in flight: every failure, 404 and 405
     included, is rendered by one of our handlers
   - `sum(authenticationResults) == authentication.total` (minus in-flight)
   - `http_clients created - closed == 0` at rest: each login closes its own client

## Adding a metric: every place it must appear

1. `app/metrics/collector.py`: a `MetricFamily` constant (`f"{METRIC_PREFIX}<name>_total"` for a
   counter, `_seconds` for a duration), documentation string, type, label names; **add it to
   `FAMILIES`** in render order.
1. Record it at the one right layer (table above).
1. `app/models/metrics.py`: a field on `MetricsModel` (or a nested model) with `title`,
   `description`, `json_schema_extra={"example": ...}`, and populate it in `from_snapshot`, casting
   explicitly (`int(...)`, `float(...)`) because the model is strict.
1. `app/docs/metrics.py`: add it to `_PROMETHEUS_EXAMPLE` (HELP, TYPE, one sample) and
   `_JSON_EXAMPLE`. `test_the_documented_prometheus_example_looks_like_the_real_payload` compares
   families with the real payload.
1. `docs/metrics.md` in the repository: a row in the right "What each metric means" table (Traffic,
   Failures, Authentication, Upstream, Internals), and the full Prometheus and JSON examples.
   `test_the_metrics_doc_describes_every_family` fails until the row exists. If an accounting rule
   changes, update "How collection works" there.
1. Tests: collector and renderer tests if the family type is new, the instrumentation test for the
   code path, and the JSON key set in `tests/unit/test_metrics_endpoints.py::test_json_format_shape`.
1. If a public dashboard or alert depends on it, say so in the PR; the Grafana dashboard linked from
   the README and `docs/metrics.md` is maintained by hand.

Renaming or removing a metric breaks dashboards and alerts: treat it as a breaking change for the
PR description even though it does not require a major version.
