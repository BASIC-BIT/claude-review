# basic-pr-review

One reusable GitHub Actions workflow that posts an advisory PR review as a single sticky PR
comment and a `basic-pr-review` commit status. It is our homegrown reviewer; today the model is
Claude. It runs on GitHub-hosted runners, needs no server, and never approves or merges.

## Adopting it

1. Copy [`templates/caller.yml`](templates/caller.yml) to `.github/workflows/basic-pr-review.yml`
   on the default branch. Keep `pull_request_target`, the event list, the permissions block and
   the pinned commit SHA. That file is the only one adoption needs.
2. Ask the role owner to add the repository's OIDC subject to the role's trust policy. The trust policy is maintained privately; it allows the token only when `job_workflow_ref` is this workflow at an approved commit SHA, `ref` is the repository's default branch (so only `pull_request_target` callers qualify) and `sub` is an enrolled repository's `pull_request` subject.
   Until then the token step fails and the sticky says the review did not complete.
3. Remove the old reviewer workflow in the same change so each PR has one publisher.
4. Optionally require the status (see [Requiring the review](#requiring-the-review)).

## When it reviews

- A ready (non-draft) PR is reviewed when it is opened, reopened, pushed to, marked ready,
  retargeted, or when a skip label is removed (`rerunOnUnskip`).
- A draft is never reviewed unless asked: add a request label (default `request-pr-review`) to
  an open PR, draft or not, and the current head is reviewed. The label is checked against the
  live PR, not the event alone. After the review publishes, the label is removed so it can be
  added again; if the review fails, the label stays so a job re-run still counts as a request.
- Other label changes and title edits do nothing and do not cancel a running review.
- A manual job re-run is the rerun path.

## Requiring the review

Each run that decides something about the current head sets a commit status with context
`basic-pr-review` on that head SHA. A commit status, not the job checks, because label and edit
events start new runs on the same head whose skipped jobs read as passing.

| Status | When |
| --- | --- |
| `success` | A review was published for this head (and `failOnImportant` does not fail it) |
| `success`, "Exempt: ..." | Skip label, bot author not allowed, every path excluded or docs-only, or `enabled: false` |
| `failure` | The review ran for this head and did not complete, or the diff is too large; re-run, or add a skip label |
| `failure`, Important | `failOnImportant` is on and the review ends with `Important findings found.`, or has no verdict line |
| none (pending) | Not reviewed yet: a draft nobody requested, a push still being reviewed, or a closed PR |

Adding a skip label after a failed review turns the status `success`. Removing a skip label
starts a fresh review (or, with `rerunOnUnskip: false`, sets it back to pending). A status is
never posted for a SHA that is no longer the PR head.

To require it, add a branch ruleset on the default branch with "Require status checks to pass"
and add the check `basic-pr-review`. Leave the source as any source; the status is posted by
GitHub Actions. PRs from forks are never reviewed and get no status, so a ruleset that requires
it blocks fork PRs.

With `failOnImportant: true`, a review whose final line is `Important findings found.` fails the
status. Adding an accept label (default `pr-review-accepted`) turns it `success` without
re-running the model: the workflow re-reads the published sticky for the current head and
trusts it only when it was posted by `github-actions[bot]` and names that head in its
completion marker. Removing the accept label re-reads the sticky the same way.

### Defaults and overrides

Both policy files are optional. Without `.github/basic-pr-review.json` (or, when that is absent,
the legacy `.github/claude-review.json`) the workflow uses:

```json
{
  "schemaVersion": 1,
  "enabled": true,
  "baseBranches": ["<the repository's default branch>"],
  "skipLabels": ["skip-pr-review", "skip-claude-review"],
  "rerunOnUnskip": true,
  "excludeGlobs": [],
  "excludeDocsOnly": false,
  "docsOnlyGlobs": ["docs/**"],
  "allowBotAuthors": false,
  "allowedBotActors": [],
  "contextMode": "allowlist",
  "contextAllowGlobs": ["**"],
  "githubLinksOnly": false,
  "instructionPaths": ["AGENTS.md", "CLAUDE.md"],
  "requestLabels": ["request-pr-review"],
  "failOnImportant": false,
  "acceptFindingsLabels": ["pr-review-accepted"]
}
```

The default branch comes from the event's repository data, and `instructionPaths` lists only
whichever of the root `AGENTS.md` and `CLAUDE.md` exist at the trusted commit. Without `REVIEW.md`
a short built-in contract applies: report correctness, security and privacy, regression,
operational, data-loss and concurrency problems at P0 to P2, each with `path:line`, impact and the
smallest fix.

To override, commit either file on the default branch. A present policy file replaces the
defaults entirely (fields are not merged), so write every required field; the example above is a
complete starting point. `requestLabels`, `failOnImportant` and `acceptFindingsLabels` are
optional and take the defaults above when left out. A present `REVIEW.md` replaces the built-in
contract.

| Field | Effect |
| --- | --- |
| `enabled` | `false` stops before any credential is requested; the status is `success` (exempt) |
| `baseBranches` | Must include the base; the base must also be the repository default branch |
| `skipLabels` | Any of these labels skips review, blocks publication and sets the status `success` |
| `rerunOnUnskip` | Removing a skip label starts a review |
| `excludeGlobs` | Git `:(glob)` pathspecs removed from the reviewed diff |
| `excludeDocsOnly` + `docsOnlyGlobs` | Skip when every old and new changed path matches; `docsOnlyGlobs` is required when enabled |
| `allowBotAuthors` | Review PRs whose author is a bot; when `false` their status is `success` (exempt) |
| `allowedBotActors` | Bot logins (or `"*"`) whose events may trigger a review |
| `contextMode` / `contextAllowGlobs` | Only `allowlist`; base-tree files outside the globs are deleted before the model runs |
| `githubLinksOnly` | Replace every non-`https://github.com` URL in the published comment |
| `instructionPaths` | Extra trusted instruction files, read from the base commit |
| `requestLabels` | Adding one reviews the current head, even on a draft |
| `failOnImportant` | Fail the status when the review reports Important findings |
| `acceptFindingsLabels` | With `failOnImportant`, any of these labels passes the status after a completed review |

A missing policy uses the defaults. A present but invalid policy (bad JSON, another
`schemaVersion`, or a failed field check) fails the job before any credential is requested.
Unknown fields are ignored. The policy, `REVIEW.md` and `instructionPaths` are read from the default-branch commit
the event ran on (`github.sha`), so a PR's edits to them are reviewed as data and take effect only
after merge.

The App-design fields and behaviors this stateless workflow does not implement: inline review
threads, a rerun command, check runs, delta review and coverage reuse, base-only invalidation,
thread auto-resolution and the work ledger.

## Versioning

Callers pin the release's full 40-character commit SHA (`@<sha> # v1.2.0`), never a tag, and the
role's trust policy lists that same SHA in `job_workflow_ref`. A SHA cannot be moved, so the code
the role trusts changes only when the trust policy does. A release means: add the new SHA to the
trust policy beside the current one, update every caller's `uses:` to it, then remove the old SHA
from trust. Both are accepted only during that rollout.

Current release: v1.2.0, `<V1.2.0 SHA, filled in after merge>` (pinned in `templates/caller.yml`).
Previous release: v1.1.0, `b1431cc7f787fc29237d42c1bc1a184dd1db5f2a`.
Callers still on v1.0.0 (`5d39ea869f2938b341459e851e294d2f8d93155e`) need `.github/claude-review.json`.

### Upgrading from v1.1.0

1. Replace `.github/workflows/claude-review.yml` with the new
   [`templates/caller.yml`](templates/caller.yml) saved as `.github/workflows/basic-pr-review.yml`.
   It changes three things, and all three are required:
   - `uses:` is now `BASIC-BIT/basic-pr-review/.github/workflows/review.yml@<v1.2.0 SHA>`.
   - The event list adds `labeled`, and the concurrency group lets `request-pr-review` replace a
     running review. If you change `requestLabels`, change that name in the group too.
   - The permissions block adds `statuses: write`. A called workflow's jobs cannot hold a
     permission the caller did not grant, so without it every status write fails the job.
2. Labels: create `request-pr-review`, `skip-pr-review` and, if you use `failOnImportant`,
   `pr-review-accepted`. `skip-claude-review` keeps working under the defaults.
3. Policy: `.github/claude-review.json` is still read when `.github/basic-pr-review.json` is
   absent. Rename it when convenient. A custom `skipLabels` is used as written, so add
   `skip-pr-review` yourself if you want it.
4. Open PRs keep their earlier sticky; the next review rewrites it with the new marker.
5. Then require the `basic-pr-review` status in a ruleset if you want merging to wait for it.

## Security model

- **One trusted workflow.** All steps that request OIDC, fetch the token, or run the model live
  in this file. It declares no inputs or secrets, so a caller cannot change the prompt, tools,
  deny rules, model, role, limits or permissions. A caller can only grant fewer permissions.
- **Event and identity gates before OIDC.** The workflow requires `pull_request_target`, an open
  PR whose head repository is the base repository (by name and ID, not the fork bit), and a base
  that is the live default branch. The PR must be non-draft, unless the event added a request
  label that the live PR still carries; that is checked again against the live PR in every
  later re-check. It then re-reads the live PR, confirms the policy commit is still on the
  default branch, applies policy, and repeats the identity checks on the token job and once more
  just before the role is assumed.
- **No PR code runs.** The workspace is the trusted base commit. The PR head is cloned into
  `pr-head/` only for `git diff` (base attributes via `--attr-source`, `--no-ext-diff`,
  `--no-textconv`) and deleted before the role is assumed. PR title and body reach the script only
  through environment variables.
- **Read-only model.** Claude runs with `--safe-mode`, `--setting-sources user`,
  `--strict-mcp-config`, the `Read` tool only (no Glob or Grep, which deny rules do not reliably cover), hooks disabled, and deny rules for shell, writes,
  agents, web, MCP, `.git`, `/proc`, `/dev`, runner temp and home credential files. Its GitHub
  token is read-only.
- **Credential handling.** AWS credentials go to step outputs only, the session lasts 15 minutes,
  the OAuth token is masked, a Claude transcript containing the token fails closed without
  logging any of it, and output containing the token or AWS secrets is not published.
- **Publication.** Only the publish job can write comments and labels, and it handles model
  output as data: decoded, scrubbed (forged `<!-- basic-` and `<!-- claude-` markers, images,
  @mentions, optional non-GitHub links), size-bounded and sent as JSON. It re-checks live PR
  state, labels an earlier review outdated instead of deleting it, and never replaces a completed
  review of the same head with a failure.
- **Statuses.** `statuses: write` is held only by the eligibility job (exemptions and the
  accept-label re-read, with no AWS credentials) and the publish job. The job that holds the AWS
  role cannot write statuses. Every status write first confirms the SHA is still the PR head.
- **Actions** are pinned by full commit SHA.

**Known limit: imitated review comments.** The sticky review is posted as `github-actions[bot]`,
and anyone who can push a branch can run a workflow on that branch that posts as the same
identity. A read-only default `GITHUB_TOKEN` does not stop this, because a workflow can request
`pull-requests: write` in its own `permissions` block. This is an accepted risk: only people and
agents who can already push to the repository can do it, the reviewer treats any previous comment
as untrusted input, and a genuine review names the exact head and base commits it covered. The
same people can also post a `basic-pr-review` status or an imitated sticky that the accept-label
path would read, so the status carries the same limit. Keep read-only defaults as hygiene. If
imitation ever becomes a practical problem, the fix is to publish under a dedicated GitHub App
identity.
