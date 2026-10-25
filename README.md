# review365

AI pull-request review for Gitea Actions and Bitbucket Pipelines.

**review365 is by [Sam Wyndham](https://github.com/mrwyndham) of [Biz365](https://biz365.com.au).**
This repo is CodeRed's port of his Bitbucket Pipelines step to Gitea, plus later additions.

## How it works

1. The PR diff goes to Claude together with the base branch's `CLAUDE.md`, `AGENTS.md` and
   `README.md`. Lockfiles, SVGs, minified JS, source maps and Unreal binary assets are left out.
   The author's claims go with it: the commit messages and, in PR mode, the PR title and
   description (10KB each, treated as data).
2. Besides bugs, a claim the code doesn't keep is a finding: the PR or a new comment promises a
   fallback, a limit or a handled case, and the code doesn't deliver it. Wording and testing
   claims are not findings.
3. Each finding must describe its attack: the attacker, the entry point, the path from source
   to sink, and the impact. Code, not the model, drops any finding that:
   - has no entry point
   - needs privileged access
   - scores under 0.8 confidence
   - uses excluded "noise" wording
4. A second, skeptical pass on Opus re-judges each surviving finding against the full files.
5. gitleaks (pinned, checksummed, offline) scans the PR's commits for secrets.
6. Each finding is posted as an inline PR comment.

## Using it

Every repo runs `review365.sh` at the **moving tag `v1`**. A repo's workflow is only a stub
with its trigger and settings: `examples/gitea.yml` for Gitea, and
`examples/bitbucket-step.yml` (kindnote's) for Bitbucket. To ship a change to every repo:

```sh
git commit -am "…" && git push
git tag -f v1 && git push -f origin v1
```

To test a change on one repo first, point that repo's stub at a branch or commit via
`REVIEW365_SRC`.

## Settings (env)

| Variable | Meaning |
|---|---|
| `CLAUDE_CODE_OAUTH_TOKEN` | Required. From `claude setup-token` (lasts about a year). |
| `REVIEW365_FOCUS` | One line describing the repo's stack. |
| `REVIEW365_CLASSES` | Replaces the default (web/money) list of bug classes. Used, for example, for Unreal C++. |
| `REVIEW365_EXTRA_RULES` | Repo-specific rules that override the "do not report" list. |
| `REVIEW365_NOISE` | Replaces the regex that drops findings for excluded wording. |
| `REVIEW365_MODEL` / `REVIEW365_VERIFY_MODEL` | Default to `sonnet` and `opus`. |
| Gitea | `API_TOKEN`, `PR`, `BASE`, `HEAD_SHA`, `REPO_URL`. The stub maps these from the event. |
| Bitbucket | `BB_REVIEW_TOKEN`, a repo access token with pullrequest:write and repository:write. |
| `REVIEW365_DRY_RUN` | `1` prints what would be posted (PR comments, commit status, issue, ntfy) instead of calling any API, and lets a local run use an already logged-in `claude`. Both modes. |

Behaviour differs by platform:
- **Gitea:** findings never fail the job. A review that couldn't run does fail it.
- **Bitbucket:** the step always passes. The `review365` commit status shows FAILED if the
  review couldn't run.

## Push mode (opt-in)

For repos that commit straight to master. It runs only when `REVIEW365_MODE=push`; with that
unset or set to anything else the script is in PR mode, unchanged. Gitea only. The stub is
`examples/gitea-push.yml` (`on: push` to master).

- **Input:** `BEFORE` and `AFTER` (the stub maps `github.event.before` and `github.sha`),
  plus `API_TOKEN` and `REPO_URL`. The review covers `git diff BEFORE AFTER`, with the same
  exclusions, prompts, filters, Opus verify pass and gitleaks (`--log-opts BEFORE..AFTER`) as
  PR mode. Project docs come from BEFORE, so a push can't rewrite the reviewer's instructions.
- **Skipped, nothing to review:**
  - `BEFORE` is all zeros (a new branch).
  - The diff is empty after the exclusions.
  - Only lockfiles changed (`flake.lock`, `pnpm-lock.yaml`, `Cargo.lock`, `go.sum`, …).
  - The push is a merge of a PR that was already reviewed. The heuristic: `AFTER` is a merge
    commit whose first parent is `BEFORE` and whose subject is Gitea's default
    `Merge pull request '<title>' (#N) from <branch> into <base>`. Squash and rebase merges,
    and repos with a custom merge-message template, don't match and are reviewed again.
- **Output:** Gitea has no commit-comment API, so a push with findings gets one issue,
  `review365: <short sha> — N finding(s)`. It links each finding to
  `REPO_URL/src/commit/AFTER/<file>#L<line>` and gives the gitleaks result. No findings means
  no issue. If `REVIEW365_NTFY_URL` is set (optionally with `REVIEW365_NTFY_TOKEN`), a
  one-line summary with the issue link is also posted there.
- **Failing:** as in PR mode on Gitea, findings never fail the job. A review that couldn't
  run does fail it, and so does an issue that couldn't be created (the token needs issue write).
- **History:** AFTER is fetched by SHA with 200 commits of history; BEFORE is fetched on its
  own if it isn't among them (a very long push, or a force-push). gitleaks then scans the
  newest 200 commits only.
