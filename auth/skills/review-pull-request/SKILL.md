---
name: review-pull-request
description: Review or re-review someone else's pesu-auth pull request for a maintainer, over as many rounds as it takes - fetch it without exposing credentials to its code, review it with review-change, post findings as suggestions the contributor can apply, check new commits against the open threads, resolve only threads that are fully done, and report what still stands between the PR and a merge. Posts only when the maintainer asks; never approves, dismisses reviews, approves workflow runs or merges. Use when a maintainer asks you to review, re-review or check on a contributor's PR.
---

# Reviewing a contributor's pull request

`review-change` is the checklist for the diff itself; this skill is everything around it on GitHub,
for a maintainer, in their session. Contributors are often students on their first pull request, so
a review should teach as well as gate: say what to change and why, and make it easy to apply.

## What you may do

| Action                                                                                                                                                                 | When                                                                                    |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| Read the PR, its threads, its checks and its code                                                                                                                      | Always                                                                                  |
| Post a review (comment or request changes), reply in a thread, resolve or reopen a thread, edit your own pending comments                                              | Only when the maintainer asks in this session. Otherwise draft it and show them         |
| Approve the PR, dismiss anyone's review, approve a workflow run, merge or add to the merge queue, push to the contributor's branch, edit the PR's title or description | Never. Tell the maintainer exactly what to click or run, and draft any message it needs |

An agent working autonomously (`.agents/pesudev-skills/auth/docs/autonomous-workflow.md`) does not
use this skill.

## 1. Fetch it without trusting it

```bash
gh pr view <n> --repo pesu-dev/auth --json title,author,headRefOid,mergeStateStatus,reviewDecision,body
git fetch upstream dev "+refs/pull/<n>/head:refs/pr/<n>"   # rerun to pick up new pushes
git worktree add --detach ../auth-pr-<n> pr/<n>             # outside your checkout: see below
git -C ../auth-pr-<n> submodule update --init
# after new pushes: git fetch upstream "+refs/pull/<n>/head:refs/pr/<n>" && git -C ../auth-pr-<n> checkout --detach pr/<n>
# when done:        git worktree remove ../auth-pr-<n> && git update-ref -d refs/pr/<n>
```

Running a pull request's code runs a stranger's code with your permissions:

- `tests/conftest.py` calls `load_dotenv()`, which looks for `.env` in `tests/` and every directory
  above it. A worktree inside your checkout hands your `TEST_*` credentials, a real PESU account, to
  the PR's tests.
- Anything it runs can read your shell's variables, `~/.config/gh` (a GitHub token that can write to
  pesu-dev) and your SSH keys.

So:

- Reading the diff is safe, and so is linting it with your own checkout's tools, which never execute
  it: `uv run ruff check ../auth-pr-<n>`, `uv run ruff format --check ../auth-pr-<n>` and
  `uvx --from mdformat==1.0.0 --with mdformat-gfm --with mdformat-frontmatter mdformat --check <files>`.
- Never run `uv sync`, `uv run` or `pre-commit` inside the PR's worktree: its `pyproject.toml` and
  `.pre-commit-config.yaml` decide what those execute.
- Let CI run it. Fork PRs get no secrets and run the reduced suite.
- If you must run it yourself, use a throwaway container that sees only the worktree, read-only:
  ```bash
  cd ../auth-pr-<n>
  docker run --rm -v "$PWD":/src:ro -w /src -e UV_PROJECT_ENVIRONMENT=/tmp/venv -e UV_CACHE_DIR=/tmp/uv \
    -e PYTHONDONTWRITEBYTECODE=1 ghcr.io/astral-sh/uv:python3.14-trixie-slim \
    uv run --frozen --all-groups pytest tests/unit -q -p no:cacheprovider
  ```
- Never run the live tests (`secret_required`) or a benchmark on someone else's code with your
  credentials.

A fork's CI runs can wait for a maintainer's approval: `gh run list --repo pesu-dev/auth --commit <sha>`
shows `action_required`. Read the diff before saying it is safe to run, above all any change to
`.github/`, `scripts/`, `.pre-commit-config.yaml`, `pyproject.toml`, `uv.lock` or the `Dockerfile`,
which CI executes. The maintainer then uses **Approve and run workflows** on the PR.

## 2. Read the conversation so far

```bash
gh api graphql -F n=<n> -f query='query($n:Int!){repository(owner:"pesu-dev",name:"auth"){pullRequest(number:$n){
  reviewThreads(first:100){nodes{id isResolved isOutdated path line comments(first:50){nodes{databaseId author{login} body}}}}
  reviews(last:20){nodes{author{login} state submittedAt commit{oid} body}}}}}'
```

- Every unresolved thread, and every request in a review's body, is open until the code shows
  otherwise.
- Weigh other reviewers' comments on their merits. Where you agree, say so briefly in their thread
  instead of repeating the point; where you do not, reply there with the reason.

## 3. Review

Load `review-change` and follow its checklist, and `security-review` when the diff touches input,
logging, auth, upstream calls or dependencies. Then check what only a pull request has:

- The **title** becomes the squash commit's message on `dev`, so it must be an accurate
  Conventional Commit for the whole change.
- The description closes the issue (`Closes #<issue>`), and the change stays within that issue.
- Exactly one version step (`bump-version`), with `uv.lock` matching.
- Anything a maintainer must do outside the code, such as setting a Render variable, goes in your
  report (section 6).

Report what matters. A blocking finding states its failure scenario; style that neither the
linters, `.agents/pesudev-skills/auth/docs/conventions.md` nor the test style in
`.agents/pesudev-skills/auth/docs/testing.md` ("Style the maintainers ask for") asks for is not a
finding.

## 4. Post, when asked

The maintainers' conventions for every review:

- **Every fix is a suggestion the contributor can apply** from the PR page. A suggestion replaces
  lines `start_line` to `line` of the PR's version of the file:
  ````markdown
  Why it should change, in a sentence or two, linking the convention it follows.

  ```suggestion
  the replacement lines, exactly as they should read
  ```
  ````
- A suggestion only works on lines inside the diff's hunks: list them with
  `gh pr diff <n> --repo pesu-dev/auth | grep -E '^(\+\+\+ |@@)'` (`+start,count` is the new side).
  For lines outside them, use a plain code block and say why ("those lines aren't part of this diff,
  so they can't be a suggestion").
- A fix spanning several places gets one suggestion per place, each saying "apply together (**Add
  suggestion to batch**)" and what breaks if only some are applied.
- Before posting, apply each suggestion literally in the worktree and run ruff, mdformat and the
  affected tests (in the container). A suggestion that fails a check costs the contributor a round.
- One review per round, with inline comments on the lines and a short summary body: what you checked,
  and what is left. Something added after posting goes in a new comment review that starts "Follow-up
  to my review above".
- Mention people only when they need to act; never write a bare `@name` for anything else.

Recipes:

```bash
# A review with inline comments: event is COMMENT or REQUEST_CHANGES (left out, it stays pending)
gh api repos/pesu-dev/auth/pulls/<n>/reviews --input review.json
#   {"commit_id": "<head sha>", "event": "REQUEST_CHANGES", "body": "...",
#    "comments": [{"path": "app/app.py", "start_line": 88, "line": 90, "side": "RIGHT", "body": "..."}]}
# Reply in an existing thread (databaseId of its first comment; posts at once)
gh api repos/pesu-dev/auth/pulls/<n>/comments/<comment id>/replies -f body='...'
# Resolve or reopen a thread (id from section 2)
gh api graphql -f query='mutation{resolveReviewThread(input:{threadId:"<id>"}){thread{isResolved}}}'
gh api graphql -f query='mutation{unresolveReviewThread(input:{threadId:"<id>"}){thread{isResolved}}}'
```

GitHub allows one pending review per person on a pull request, so creating a review fails while the
maintainer has a draft. Ask whether to add to their draft: find its `node_id` with
`gh api repos/pesu-dev/auth/pulls/<n>/reviews --jq '.[] | select(.state=="PENDING")'`, add comments
with the GraphQL mutations `addPullRequestReviewThread` (new thread) and
`addPullRequestReviewThreadReply` (reply), passing that `pullRequestReviewId`, and leave submitting
it to them.

## 5. Re-review new commits

```bash
gh api repos/pesu-dev/auth/pulls/<n>/reviews --jq '[.[] | select(.user.login=="<you>")][-1].commit_id'
git diff <last reviewed sha> pr/<n>                                               # no rebase since
diff <(git diff upstream/dev...<last reviewed sha>) <(git diff upstream/dev...pr/<n>)   # after a rebase
```

- Check every open thread against the new head. Resolve a thread only when everything it asked for
  is done. If part is still open, keep it open and reply with what is left; reopen a thread someone
  resolved early, because a resolved thread is collapsed and the contributor will miss it.
- Commits named "Apply suggestion from ..." with the reviewer as co-author come from the contributor
  applying suggestions on GitHub. That is how it should work.
- Review the new code as in section 3: fixes can bring their own problems.

## 6. Ready to merge?

The `dev` ruleset (`gh api repos/pesu-dev/auth/rules/branches/dev`) requires:

- **Greenlight - CI checks** passing on the head commit, after any approval the fork's runs needed;
- every review thread resolved;
- an approving review from a code owner (`.github/CODEOWNERS`) given after the last push;
- no standing "changes requested" from anyone with write access: they re-review, or a maintainer
  dismisses it (draft the dismissal message, naming what was addressed);
- merging through the merge queue, as a squash with the PR's title.

Before a maintainer merges, also list:

- **Render settings:** a new environment variable or secret is set on pesuauth-dev before the merge,
  because a merge deploys staging at once, and on pesuauth before the next release.
- **Release order:** Deploy to Production releases `dev`'s latest commit, so work already on `dev`
  that should ship without this change is released first.
- **Agent docs:** if the change makes anything in `.agents/pesudev-skills/auth/` wrong, a pull request
  to pesu-dev/skills (`update-docs`).

## Report

```markdown
## Verdict
ready to merge | changes required | waiting on the maintainer (<what>)

## Outstanding (most severe first)
1. `<file>:<line>`: <what is still wrong or missing>. Thread: <link, or "new">.

## Done since the last round
- <request>: <how it was addressed>

## Before merging (maintainer)
- <CI approval, approval, dismissals, Render settings, release order, agent docs>
```

Say plainly when nothing is left. Never report a check as passing that you did not see pass.
