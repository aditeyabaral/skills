---
name: add-metric
description: Add or change a pesu-auth metric - MetricFamily in the registry, recording at the right layer with bounded labels, the JSON MetricsModel view, both documentation examples, the docs/metrics.md row and tests - while keeping the accounting invariants true. Use for any new counter, gauge or summary, or a change to an existing one.
---

# Adding a metric

Read `.agents/pesudev-skills/auth/docs/metrics.md` (layers, rules, invariants) first.

## 1. Design it

- **Question it answers.** If an existing family already answers it (often `errors_total{type}` or
  `upstream_*`), do not add a second one; one fact, one place.
- **Type.** `counter` (only increases), `gauge` (current value), or `summary` (latency: `_sum` and
  `_count` via `observe`).
- **Name.** `pesu_auth_<noun>_total` for counters, `pesu_auth_<noun>_seconds` for durations.
- **Labels.** Only from a closed set the code controls. Write the full set of possible values down.
  If any value can come from a caller or from upstream text, clamp it to a known set with an
  `other` bucket first.
- **Layer.** Middleware (every request), exception handler (error class), route or lifespan
  (`app/app.py`), or upstream client (`app/pesu.py`). Pick exactly one.

## 2. Registry: `app/metrics/collector.py`

```python
FOO_TOTAL = MetricFamily(
    f"{METRIC_PREFIX}foo_total",
    "One sentence: what is counted, and by what label.",
    "counter",
    ("outcome",),
)
```

Add it to `FAMILIES` next to related families (that tuple is the render order and the single source
of HELP/TYPE).

## 3. Record it

`metrics.increment(FOO_TOTAL, outcome="success")` in `app/app.py` (module singleton `metrics`), or
`self._metrics.increment(...)` in `app/pesu.py`. For a summary, `observe(FAMILY, seconds, **labels)`
with a `time.perf_counter()` duration. Labels must exactly match the declared names or the collector
raises `ValueError`. Recording inside a `finally` is how the code guarantees exactly-once on every
exit path; do the same when the path can raise or be cancelled.

## 4. JSON view: `app/models/metrics.py`

Add a field to `MetricsModel` (camelCase on the wire automatically) with `title`, `description`
and `json_schema_extra={"example": ...}`, then populate it in `from_snapshot`. For a single-label
counter use `_counts(snapshot, FOO_TOTAL.name, "outcome")`. Cast explicitly (`int(...)`), because
the model is strict. Import the constant at the top of the file with the others.

## 5. Documentation

- `app/docs/metrics.py`: HELP, TYPE and one sample line in `_PROMETHEUS_EXAMPLE` (in `FAMILIES`
  order), and the key in `_JSON_EXAMPLE`.
- The repository's `docs/metrics.md`: a row in the right "What each metric means" table (Traffic,
  Failures, Authentication, Upstream, Internals), the full Prometheus example and the full JSON
  example. `test_the_metrics_doc_describes_every_family` fails until the row exists. If an accounting
  rule changes, update "How collection works" there.

## 6. Tests

- The recording path: assert the exact series after the action, using a fresh collector
  (`MetricsCollector(clock=lambda: 1000.0)` injected into `PESUAcademy`, or
  `monkeypatch.setattr("app.app.metrics", MetricsCollector())` for the app). See
  `tests/unit/test_metrics_instrumentation.py` and `tests/unit/test_metrics_endpoints.py`.
- Update the key set in `tests/unit/test_metrics_endpoints.py::test_json_format_shape`.
- Prometheus rendering of the new family appears (`tests/unit/test_metrics_prometheus.py` style).
- `uv run pytest tests/unit/test_openapi_docs.py -q` (the Prometheus example is compared with a real
  payload).

## 7. Verify by hand

Run the app (`run-locally`), exercise the path, then:

```bash
curl -s localhost:5000/metrics | grep foo_total
curl -s "localhost:5000/metrics?fmt=json" | python -m json.tool | grep -A3 foo
```

Check the invariants in `.agents/pesudev-skills/auth/docs/metrics.md` still hold on the numbers you see. Renaming or removing a
metric breaks dashboards: call it out in the PR.
