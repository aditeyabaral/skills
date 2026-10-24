---
name: update-docs
description: Bring pesu-auth documentation in step with behaviour - README sections and examples, OpenAPI examples in app/docs, CONTRIBUTING, .env.example, docstrings and why-comments - and format it with mdformat. Use after any behaviour, configuration, metrics, workflow or tooling change, or for documentation-only fixes.
---

# Updating documentation

## Where each kind of fact is documented

| Fact                                           | Places                                                                                                                                                                                   |
| ---------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Endpoints, parameters, responses, status codes | README "How to use the PESUAuth API" (endpoint table, per-route sections), `app/docs/<route>.py`, route docstrings in `app/app.py`                                                       |
| Request/response/profile fields                | README tables (Request Parameters, Response Object, `ProfileObject`), integration examples at the end of the README, model `Field` descriptions and examples, `app/docs/authenticate.py` |
| Error messages                                 | exception default messages, `app/docs/*` examples, README Responses tables                                                                                                               |
| Metrics                                        | README "`/metrics`" (meaning tables, full Prometheus and JSON examples, accounting rules), `app/docs/metrics.py`, `MetricFamily` documentation strings, `MetricsModel` descriptions      |
| Configuration (`METRICS_TOKEN`, CLI flags)     | README "Protecting the endpoint" and run instructions, `main()` help text, `.env.example` for test variables                                                                             |
| Setup, tests, workflow, labels                 | `.github/CONTRIBUTING.md`, `.github/PULL_REQUEST_TEMPLATE.md`                                                                                                                            |
| Deployment and monitoring links                | README top sections, CONTRIBUTING "Deployment Environment"                                                                                                                               |
| Why code is the way it is                      | comments and docstrings next to the code                                                                                                                                                 |

## Steps

1. List what the change alters. For each item, search every place above:
   `rg -n "<field|message|metric|flag>" README.md .github app`.
1. Edit each place. Keep the existing voice: direct, precise, explains why, no marketing. README
   examples must be copy-pasteable and match real output (run the request and paste the real shape,
   with personal values replaced by the placeholder values already used: `PES1201800001`,
   `John Doe`, `johndoe@gmail.com`, `1234567890`).
1. Swagger examples in `app/docs/` must validate against their models:
   `uv run pytest tests/unit/test_openapi_docs.py -q`.
1. Format Markdown: `uv run pre-commit run mdformat --all-files` (it rewrites tables and list
   numbering; commit what it produces).
1. Look at Swagger (`run-locally`, then `http://localhost:5000/`) for any route you touched.

## Rules

- Document only what the code does. If the docs and code disagree and the task is not about that,
  note it in `.agents/pesudev-skills/auth/docs/known-issues.md` in pesu-dev/skills or the PR rather than guessing.
- Do not reorganise README sections as a side effect of another change.
- Never include real credentials, tokens or personal data in examples.
- Doc-only changes still bump the version (every PR does): `docs:` commit type.
