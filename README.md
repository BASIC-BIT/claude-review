# claude-review

One reusable GitHub Actions workflow that posts an advisory Claude review as a single sticky PR
comment. It runs on GitHub-hosted runners, needs no server, and never approves or merges.

## Adopting it

1. Copy [`templates/caller.yml`](templates/caller.yml) to `.github/workflows/claude-review.yml` on
   the default branch. Keep `pull_request_target`, the permissions block and the pinned commit SHA.
2. Commit `.github/claude-review.json` on the default branch:

   ```json
   {
     "schemaVersion": 1,
     "enabled": true,
     "baseBranches": ["main"],
     "skipLabels": ["skip-claude-review"],
     "rerunOnUnskip": true,
     "excludeGlobs": [],
     "excludeDocsOnly": true,
     "docsOnlyGlobs": ["docs/**"],
     "allowBotAuthors": false,
     "allowedBotActors": [],
     "contextMode": "allowlist",
     "contextAllowGlobs": ["**"],
     "githubLinksOnly": false,
     "instructionPaths": ["AGENTS.md"]
   }
   ```

3. Optionally commit `REVIEW.md` (repository-specific calibration) at the root.
4. Ask the role owner to add the repository's OIDC subject to the role's trust policy. The trust policy is maintained privately; it allows the token only when `job_workflow_ref` is this workflow at an approved commit SHA, `ref` is the repository's default branch (so only `pull_request_target` callers qualify) and `sub` is an enrolled repository's `pull_request` subject.
   Until then the token step fails and the sticky says the review did not complete.
5. Remove the old reviewer workflow in the same change so each PR has one publisher.

| Field | Effect |
| --- | --- |
| `enabled` | `false` stops before any credential is requested |
| `baseBranches` | Must include the base; the base must also be the repository default branch |
| `skipLabels` | Any of these labels skips review and blocks publication |
| `rerunOnUnskip` | Removing a skip label starts a review |
| `excludeGlobs` | Git `:(glob)` pathspecs removed from the reviewed diff |
| `excludeDocsOnly` + `docsOnlyGlobs` | Skip when every old and new changed path matches; `docsOnlyGlobs` is required when enabled |
| `allowBotAuthors` | Review PRs whose author is a bot |
| `allowedBotActors` | Bot logins (or `"*"`) whose events may trigger a review |
| `contextMode` / `contextAllowGlobs` | Only `allowlist`; base-tree files outside the globs are deleted before Claude runs |
| `githubLinksOnly` | Replace every non-`https://github.com` URL in the published comment |
| `instructionPaths` | Extra trusted instruction files, read from the base commit |

Missing or invalid policy fails the job before any credential is requested. Unknown fields are
ignored. The policy, `REVIEW.md` and `instructionPaths` are read from the default-branch commit
the event ran on (`github.sha`), so a PR's edits to them are reviewed as data and take effect only
after merge.

The App-design fields and behaviors this stateless workflow does not implement: inline review
threads, the `/claude-review rerun` command, check runs, delta review and coverage reuse, base-only
invalidation, thread auto-resolution and the work ledger. A manual job re-run is the rerun path.

## Versioning

Callers pin the release's full 40-character commit SHA (`@<sha> # v1.0.0`), never a tag, and the
role's trust policy lists that same SHA in `job_workflow_ref`. A SHA cannot be moved, so the code
the role trusts changes only when the trust policy does. A release means: add the new SHA to the
trust policy beside the current one, update every caller's `uses:` to it, then remove the old SHA
from trust. Both are accepted only during that rollout.

## Security model

- **One trusted workflow.** All steps that request OIDC, fetch the token, or run Claude live in
  this file. It declares no inputs or secrets, so a caller cannot change the prompt, tools, deny
  rules, model, role, limits or permissions. A caller can only grant fewer permissions.
- **Event and identity gates before OIDC.** The workflow requires `pull_request_target`, an open
  non-draft PR whose head repository is the base repository (by name and ID, not the fork bit),
  and a base that is the live default branch. It then re-reads the live PR, confirms the policy
  commit is still on the default branch, applies policy, and repeats the identity checks on the
  token job and once more just before the role is assumed.
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
- **Publication.** Only the publish job can write, and it handles model output as data: decoded,
  scrubbed (forged `<!-- claude-` markers, images, @mentions, optional non-GitHub links),
  size-bounded and sent as JSON. It re-checks live PR state, labels an earlier review outdated
  instead of deleting it, and never replaces a completed review of the same head with a failure.
- **Actions** are pinned by full commit SHA.

**Recommended repository setting.** Set GitHub Actions default workflow permissions to read-only
in every consumer repository or organization. Otherwise any workflow on a same-repository branch
gets a writable `GITHUB_TOKEN` and can post comments as `github-actions[bot]` that look exactly
like this reviewer's sticky. This is a settings change BASIC makes; the workflow cannot enforce it.
