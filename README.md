# pesu-dev/skills

Agent instructions, skills and roles for pesu-dev projects, kept in one repository. Each project
repo mounts this repo as a git submodule, so any coding agent opened in that project (Codex, Claude
Code, Copilot, Cursor, Gemini and others) finds the project's material straight after cloning.
Contributors never clone this repo themselves.

Everything is plain Markdown in agent-neutral formats: `AGENTS.md` for instructions,
[Agent Skills](https://agentskills.io) (`SKILL.md`) for skills, and plain Markdown files for roles.
This repo has no scripts to run and nothing in it is generated.

## Layout

```
common/                        shared across pesu-dev
├── AGENTS.md                  org-wide conventions, to copy from into project AGENTS.md files
└── skills/<name>/SKILL.md     shared skills; projects link to the ones they want
<project>/                     one folder per repo, named after it, owned by its maintainers
├── AGENTS.md                  the project's instructions: the entry point agents read
├── skills/
│   ├── <name>/SKILL.md                       the project's own skills
│   └── <name> -> ../../common/skills/<name>  links to common skills it uses
├── docs/<topic>.md            optional: reference material agents read on demand
└── agents/<role>.md           optional: role definitions (planner, reviewer, ...)
.github/                       CODEOWNERS and the two workflows (see Checks)
.pre-commit-config.yaml        the checks CI runs
```

**`auth/` is the reference project.** It has everything a project folder can have: an `AGENTS.md`,
ten reference docs, ten roles and 24 skills that together let an agent take a change from an issue
to an open pull request on its own. Start a new project by reading it and copying the parts that
fit.

## How a project repo uses this

```
<project repo>/
├── .agents/pesudev-skills/   submodule: this repo
├── AGENTS.md           -> .agents/pesudev-skills/<project>/AGENTS.md
├── .agents/skills      -> pesudev-skills/<project>/skills
└── .github/agents/<role>.agent.md   optional: generated Copilot wrappers for the roles
```

The links use the two standard, provider-neutral locations: `AGENTS.md`, which every agent reads,
and `.agents/skills`, the Agent Skills location. No tool-specific links (`.claude/`, `.cursor/`,
...) are added. An agent that does not load skills from `.agents/skills` natively still finds every
skill through the index in `AGENTS.md`, which gives each skill's path and when to use it. Docs and
roles need no links: `AGENTS.md` and the skills point agents at
`.agents/pesudev-skills/<project>/docs/` and `.agents/pesudev-skills/<project>/agents/` directly.

The links point at folders, so they never change. A new skill, doc or instruction reaches a project
when its submodule is bumped.

### Roles as Copilot custom agents

`.github/agents/` is the one tool-specific path, because a link cannot work there. On GitHub.com,
Copilot lists custom agents only from real `.github/agents/<name>.agent.md` files on the repo's
default branch, and GitHub does not read files through a submodule (VS Code reads the local folder,
but the GitHub.com picker and the cloud agent do not). A project that wants its roles in Copilot's
agent picker keeps a thin wrapper per role there: the role's frontmatter, copied verbatim, and a short body
telling the agent to read the full role file in `.agents/pesudev-skills/<project>/agents/`. The role
file stays the only definition.

The wrappers are generated and drift-checked in the project repo, never written by hand: `auth`
does it with `scripts/sync_agents.py` and a pre-commit hook that runs the script with `--check`. A
submodule bump that changes a role's frontmatter fails that check until the wrappers are
regenerated (see "Changing anything"). Copilot's cloud agent also needs the submodule checked out, which the project's
`.github/workflows/copilot-setup-steps.yml` does.

## Writing a project folder

### `AGENTS.md`

The one file every agent reads in full, so keep it to a few hundred lines and point to the rest.
`auth/AGENTS.md` shows the shape:

1. What the project is and where it runs.
1. Where the agent material lives (skills, docs, roles) and how short paths resolve.
1. Quick-reference commands: set up, run, test, lint, build.
1. A map of the repository.
1. Rules that always apply, each pointing to the doc with the detail.
1. The contribution workflow (branches, commits, PRs, versioning).
1. What an agent may do on its own and when it must stop.
1. An index of skills, roles and docs, each with "use when".

Only include what an agent cannot work out from the code. Copy what applies from `common/AGENTS.md`
into it: an agent reads a single `AGENTS.md`, so org-wide rules only apply if they are in the
project's file (or it explicitly tells agents to read `.agents/pesudev-skills/common/AGENTS.md`).

### Skills

A skill is a folder with a `SKILL.md`: YAML frontmatter, then the instructions an agent follows once
it loads the skill.

```markdown
---
name: run-tests
description: What the skill does and when to use it. Agents read only this to decide whether to load it, so name the task and the situations that call for it.
---

# Running tests

Numbered steps with the exact commands and file paths, checklists, and how to verify the result.
```

- `name` must match the folder name: lowercase letters, digits and hyphens. Agents skip a skill
  whose `name` or `description` is missing, without any error.
- Write for the task, not the topic: `add-endpoint`, `fix-ci`, `bump-version`. One skill should be
  the orchestrator that runs a whole change end to end and calls the others (`auth`'s
  `implement-feature`).
- Give exact commands and repository paths, the checks that prove the work is done, and what the
  agent must never do.
- Scripts, templates or reference files a skill needs go next to its `SKILL.md`.
- Keep Python code blocks formatted the way the project's ruff would format them. Ruff formats code
  blocks in Markdown, and inside a project repo it sees the submodule, so a badly formatted example
  makes a local `ruff format --check .` fail there.

### Docs

Longer reference material, in `<project>/docs/<topic>.md`: architecture, conventions, testing, API
contract, CI/CD, security, and so on. Agents open a doc when `AGENTS.md` or a skill points them to
it, so every doc should be linked from somewhere. A `known-issues.md` listing things in the project
that are wrong or misleading stops agents from copying them.

### Roles

A role describes one phase of autonomous work: planning, implementing, testing, reviewing,
releasing. It is plain Markdown with `name` and `description` frontmatter, so it works with any
agent: tools that can start sub-agents run the role as one, and other tools follow the file
themselves for that phase. `AGENTS.md` or a skill says which role to use when.

```markdown
---
name: reviewer
description: What the role does and when to use it. Say whether it is read-only.
tools: ["read", "search", "execute", "github/*"]
---

# Reviewer

## Inputs
## Do
## Output
## Never
## Stop and escalate when
```

`tools` is optional and only for read-only roles. It uses GitHub Copilot's tool aliases (`read`,
`search`, `execute`, `edit`, `web`, `agent`, `github/*`), which Copilot applies to the role's
custom agent (see [Roles as Copilot custom agents](#roles-as-copilot-custom-agents)); other tools
ignore it. Leaving `edit` out takes away Copilot's file-editing tools, but the shell (`execute`)
can still write files, so the role's own "Never" section remains the actual rule. Leave `tools` out
entirely for read-write roles: no `tools` means all tools.

### Paths

Inside a project repo, this repo is at `.agents/pesudev-skills/`. Skills, docs and roles refer to each
other and to the project's code by paths from the project repo's root (for example
`.agents/pesudev-skills/auth/docs/testing.md` or `app/pesu.py`), never by relative paths: a skill read
through the `.agents/skills` link would resolve them to the wrong place.

## Common skills

Link a common skill into a project's `skills/` folder to use it; later changes to it then reach the
project automatically:

```bash
ln -s ../../common/skills/pesu-review <project>/skills/pesu-review
```

A newly added common skill is not linked anywhere until a project's maintainers link it. Do not
link a common skill that a project skill already covers (`auth` has its own `review-change`, for
example): agents would have two competing skills for the same task.

## Adding a project

1. Here: create `<project>/` with an `AGENTS.md` and a `skills/` folder containing at least one
   skill or common-skill link (git does not track empty folders), and add the maintainers to
   `.github/CODEOWNERS`.

1. In the project repo, on a branch, add the submodule and the links:

   ```bash
   git submodule add https://github.com/pesu-dev/skills.git .agents/pesudev-skills
   ln -s .agents/pesudev-skills/<project>/AGENTS.md AGENTS.md
   ln -s pesudev-skills/<project>/skills .agents/skills
   git add .gitmodules .agents AGENTS.md
   ```

1. Still in the project repo, add `.github/dependabot.yml`, so updates to this repo arrive as pull
   requests that bump the submodule:

   ```yaml
   version: 2
   updates:
     - package-ecosystem: gitsubmodule
       directory: /
       schedule:
         interval: daily
       target-branch: dev
   ```

   Workflows that only accept pull requests from forks, or that require a version bump, need an
   exception for `dependabot[bot]`.

1. Add the cloning instructions below to the project's README or `CONTRIBUTING.md`.

1. Optional, if the project has roles: generate Copilot wrappers for them as described in
   [Roles as Copilot custom agents](#roles-as-copilot-custom-agents), copying `auth`'s
   `scripts/sync_agents.py`, its `sync-agents` pre-commit hook, `submodules: true` on the
   checkout of the CI job that runs the hooks, and `.github/workflows/copilot-setup-steps.yml`.

## Cloning a project

```bash
git clone --recurse-submodules https://github.com/<you>/<project>.git
# already cloned without it:
git submodule update --init
```

Without the submodule, the links point at nothing, so agents see no skills and no instructions.

On Windows, turn on Developer Mode and run `git config --global core.symlinks true` **before**
cloning, or git checks the links out as plain text files.

## Changing anything

Open a pull request here, not in the project repo: everything under `.agents/pesudev-skills` is a
pinned checkout of this repo. Like the pesu-dev projects, pull requests come from a branch of your
fork that is not `main`, and target `main` (this repo has no `dev` branch, since nothing here is
deployed). Once it merges, Dependabot opens a pull request in each project to
bump the submodule. To try an unmerged change, run `git -C .agents/pesudev-skills checkout <branch>`
inside the project.

If your change adds or removes a role or edits a role's frontmatter (`name`, `description`,
`tools`), Dependabot's pull request fails the `sync-agents` check in every project that generates
Copilot custom agents, and Dependabot cannot regenerate them. Follow up in each such project with a
pull request from your fork that bumps the submodule and regenerates the agents;
`auth/docs/ci-cd.md` ("Dependabot submodule bumps") has the steps. Changes to a role's body need nothing:
the wrappers only point at it.

## Checks

Two workflows run on every pull request:

| Workflow                                                   | Checks                                                                                                                                                                                                                                             |
| ---------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Enforce PR Origin Policy (`.github/workflows/source.yaml`) | The pull request comes from a fork, not from its `main`, and targets `main`                                                                                                                                                                        |
| Pre-Commit Checks (`.github/workflows/pre-commit.yaml`)    | The hooks in `.pre-commit-config.yaml`: file hygiene, links that point nowhere or were checked out as plain files, Markdown formatting (`mdformat`, keeping YAML frontmatter intact), and ruff formatting of Python code blocks at line length 120 |

Nothing needs installing to contribute: CI runs the hooks, and a failure shows the diff to apply.
To run them locally anyway: `uvx pre-commit run --all-files`.

## Who owns what

`common/` and this README are shared. Each project folder belongs to that project's maintainers
(see `.github/CODEOWNERS`): they decide what its `AGENTS.md` says and which skills, docs and roles
it has.
