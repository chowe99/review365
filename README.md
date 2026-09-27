# review365

AI pull-request review for Gitea Actions and Bitbucket Pipelines.

**review365 is by [Sam Wyndham](https://github.com/mrwyndham) of [Biz365](https://biz365.com.au).**
This repo is CodeRed's port of his Bitbucket Pipelines step to Gitea, plus later additions.

## How it works

1. The PR diff goes to Claude together with the base branch's `CLAUDE.md`, `AGENTS.md` and
   `README.md`. Lockfiles, SVGs, minified JS, source maps and Unreal binary assets are left out.
2. Each finding must describe its attack: the attacker, the entry point, the path from source
   to sink, and the impact. Code, not the model, drops any finding that:
   - has no entry point
   - needs privileged access
   - scores under 0.8 confidence
   - uses excluded "noise" wording
3. A second, skeptical pass on Opus re-judges each surviving finding against the full files.
4. gitleaks (pinned, checksummed, offline) scans the PR's commits for secrets.
5. Each finding is posted as an inline PR comment.

## Using it

Every repo runs `review365.sh` at the **moving tag `v1`**. A repo's workflow is only a stub
with its trigger and settings; see `examples/`. To ship a change to every repo:

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

Behaviour differs by platform:
- **Gitea:** findings never fail the job. A review that couldn't run does fail it.
- **Bitbucket:** the step always passes. The `review365` commit status shows FAILED if the
  review couldn't run.
