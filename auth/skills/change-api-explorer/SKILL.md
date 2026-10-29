---
name: change-api-explorer
description: Change pesu-auth's API Explorer, the page at / (app/templates/playground.html) - its layout, styling, controls, badges or behaviour - keeping it one self-contained file with no build step or third-party code, built from /openapi.json, safe with server data and credentials, themed in light and dark, tested in tests/unit/test_playground.py and checked in a real browser. Use for any change to the page at /, including the follow-ups in issue 248.
---

# Changing the API Explorer

The Explorer is `app/templates/playground.html`: one HTML file with its CSS and JavaScript inline,
served as is by the `/` route in `app/app.py` (`FileResponse`). It has no build step, loads nothing
from another site (its only external link is to the GitHub repository), and builds its sidebar,
forms and response examples in the browser from `/openapi.json`.

**First check whether the Explorer needs changing at all.** A new route, field, example or
description reaches it through the schema: change the route's docs (`add-endpoint`,
`change-api-models`) and the Explorer shows it. It lists GET, POST, PUT, PATCH and DELETE
operations, and adds Swagger UI (`/docs`) as the last entry.

## Where things are

Search by name; line numbers move.

| Part                                 | Name                                                                                                |
| ------------------------------------ | --------------------------------------------------------------------------------------------------- |
| Theme applied before the first paint | the `<script>` in `<head>`, reading `localStorage` key `pesuauth-theme`                             |
| Colours and spacing                  | the custom properties in `:root` and `:root[data-theme="dark"]`                                     |
| Navbar badges                        | `#version` (from the spec) and `#environment` (from `/health`)                                      |
| Page state and helpers               | `state`, `$` (`getElementById`), `esc` (HTML escaping), `toast`                                     |
| Reading the schema                   | `resolveSchema`, `schemaType`, `exampleForSchema`, `requestExample`                                 |
| Drawing an operation                 | `selectEndpoint`, `renderFields`, `renderBody`, `renderQuery`, `renderResponses`, `renderMarkdown`  |
| The request                          | `buildRequest`, `currentCurl`, and `$("requestForm").onsubmit`, which sends it and shows the result |
| Loading                              | `load()` (the spec, then the sidebar) and `loadEnvironment()` (the badge), both called at the end   |

## Rules

- **Vanilla only.** No library, framework, CDN, web font or build step; everything stays in this
  one file.
- **Colours come from the tokens.** Reuse an existing one where it fits (`--success`, `--warning`,
  `--danger`, `--get`, `--post`, `--muted`, ...). A new token is defined in both `:root` and
  `:root[data-theme="dark"]`.
- **Server data is text.** A value from a response (`/health`, `/authenticate`, `/metrics`) or the
  schema is set with `textContent`. Where the page has to build markup around it, it goes through
  `esc()` first, as `responseHtml()`, `highlightJson()` and `inline()` do. Never put an unescaped
  value into `innerHTML`.
- **Credentials stay in their fields.** The password and the bearer token are never written to
  `localStorage`, `sessionStorage`, the URL, the console or a toast, and never copied anywhere except
  by **Copy request** when the user asks. `localStorage` holds the theme and nothing else
  (`.agents/pesudev-skills/auth/docs/security.md`).
- **Each request the page makes on load costs one more request per visit in `/metrics`.** Today a
  visit fetches `/`, `/openapi.json`, `/docs` and `/health`, and the browser asks for `/favicon.ico`
  (issue 248). Add a fetch only when the page needs it,
  never make the sidebar wait for it, and give it its own `catch`, so a failure only hides its own
  part: `loadEnvironment()` is the pattern.
- **Older or failing servers.** If a response lacks a field the page wants, the element stays
  hidden; the rest of the page works.
- **Accessibility.** A new control is a `<button>` with an `aria-label` (and `type="button"` inside
  the request form, so it does not submit it); a toggle publishes `aria-pressed` or `aria-checked`, as
  the theme switcher does; everything works from the keyboard.
  Animations respect `prefers-reduced-motion`, and the layout has breakpoints at 1040px and 700px.
- **Formatting.** No formatter runs on this file (only the whitespace hooks): match the two-space
  indentation and the style of the code around your change.

## Tests

`tests/unit/test_playground.py` reads the HTML as text; nothing runs the JavaScript. Assert the parts
that carry the behaviour, the way the existing tests do:

- markup that must exist, with its id and initial state
  (`test_the_navbar_has_a_hidden_environment_badge`);
- the code that must be there, and code that must not appear inside a function, such as `innerHTML`
  (`test_the_environment_badge_is_filled_from_health`, `test_a_failing_health_does_not_break_the_explorer`);
- CSS rules that use the right tokens (`test_each_environment_has_its_own_colour`, parametrized
  through the `environment` fixture in `tests/unit/conftest.py`).

The file is not Python, so coverage does not change, but every change still comes with tests
(`write-tests`).

## Check it in a browser

The tests cannot run the page, so look at it:

```bash
uv run python -m app.app --debug     # then open http://localhost:5000/
```

- the change in the light, dark and system themes, and at a narrow width;
- send a request, and use **Copy request** and the response's **Copy** (try `/health` or `/metrics`:
  `/authenticate` signs in to PESU Academy for real);
- the console shows no errors, and the network tab shows only the requests you expect;
- make the endpoint your change reads fail for a moment (return an error from it locally, without
  committing that), and check that only your part disappears.

With a browser tool, take screenshots of both themes for the pull request.

## Documentation

- The README's API Explorer paragraph describes what a visitor sees; keep it exact (issue 248's
  first item is a case where it drifted).
- If the page starts relying on an endpoint or field, that field becomes part of the API contract:
  note it in `.agents/pesudev-skills/auth/docs/api-contract.md` with a pull request to pesu-dev/skills.
- Open follow-ups for the Explorer are in issue 248.
