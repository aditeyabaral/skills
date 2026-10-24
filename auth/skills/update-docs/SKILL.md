---
name: update-docs
description: Bring pesu-auth documentation in step with behaviour - README sections and examples, OpenAPI examples in app/docs, CONTRIBUTING, .env.example, docstrings and why-comments - and format it with mdformat. Use after any behaviour, configuration, metrics, workflow or tooling change, or for documentation-only fixes.
---

# Updating documentation

## Where each kind of fact is documented

| Fact                                               | Places                                                                                                                                                                                                                                                     |
| -------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Endpoints, parameters, responses, status codes     | README "How to use the PESUAuth API" (endpoint table, per-route sections), `app/docs/<route>.py`, route docstrings in `app/app.py`                                                                                                                         |
| Request/response/profile fields                    | README tables (Request Parameters, Response Object, `ProfileObject`), the full request and response under "Integrating your application", model `Field` descriptions and examples, `app/docs/authenticate.py`                                              |
| Errors and messages                                | exception default messages, `app/docs/*` examples, README Responses tables, the error table in `docs/architecture.md`                                                                                                                                      |
| Metrics                                            | `docs/metrics.md` (meaning tables, accounting rules, full Prometheus and JSON examples, protecting and scraping), README "`/metrics`" (parameters and responses), `app/docs/metrics.py`, `MetricFamily` documentation strings, `MetricsModel` descriptions |
| Configuration (CLI flags, `PORT`, `METRICS_TOKEN`) | The configuration table in `docs/development.md`, README run instructions, `_build_arg_parser()` help text, `.env.example` for test variables                                                                                                              |
| How the service works                              | `docs/architecture.md`                                                                                                                                                                                                                                     |
| Setup, tooling, tests, benchmarks                  | `docs/development.md`, `docs/testing.md`                                                                                                                                                                                                                   |
| Contribution process, rules, labels                | `.github/CONTRIBUTING.md` (process and policy only; reference material goes in `docs/`), `.github/PULL_REQUEST_TEMPLATE.md`                                                                                                                                |
| Deployment and monitoring                          | `docs/deployment.md`, README top sections, CONTRIBUTING "Deployment Environment"                                                                                                                                                                           |
| Why code is the way it is                          | comments and docstrings next to the code                                                                                                                                                                                                                   |

## Steps

1. List what the change alters. For each item, search every place above:
   `rg -n "<field|message|metric|flag>" README.md docs .github app`.
1. Edit each place. Keep the existing voice: direct, precise, explains why, no marketing. README
   examples must be copy-pasteable and match real output (run the request and paste the real shape,
   with personal values replaced by the placeholder values already used: `PES1202000001`,
   `PES1UG20CS001`, `John Doe`, `johndoe@gmail.com`, `1234567890`, `2002-01-31`).
1. Swagger examples in `app/docs/` must validate against their models:
   `uv run pytest tests/unit/test_openapi_docs.py tests/unit/test_docs.py -q`.
1. Format Markdown: `uv run pre-commit run mdformat --all-files` (it rewrites tables and list
   numbering; commit what it produces).
1. Look at the API Explorer (`http://localhost:5000/`) and Swagger (`/docs`) for any route you
   touched (`run-locally`).

## Rules

- Document only what the code does. If the docs and code disagree and the task is not about that,
  note it in `.agents/pesudev-skills/auth/docs/known-issues.md` in pesu-dev/skills or the PR rather than guessing.
- Do not reorganise README sections as a side effect of another change.
- Never include real credentials, tokens or personal data in examples.
- `tests/unit/test_docs.py` checks the error and configuration tables and every path the `docs/`
  pages name; run it with `test_openapi_docs.py` after any docs change.
- In anything a caller reads, do not mention "the mobile API", and do not put examples inside field
  descriptions: the schema's `example` covers them.
- Doc-only changes still bump the version (every PR does): `docs:` commit type.
