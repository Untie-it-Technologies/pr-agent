# Implementing Incremental PR Review in GitHub Actions

This guide explains how to wire up the incremental PR review feature from this fork
of PR-Agent into a GitHub Actions workflow. No external infrastructure (Redis, DB)
is required — state is stored in hidden PR comments.

---

## How it Works

| Trigger | What runs |
|---|---|
| PR **opened / reopened / ready_for_review** | Full review of all changed files |
| PR **synchronize** (new commit pushed) | Incremental review of only the new diff since last review |

The last-reviewed HEAD SHA is stored as a hidden HTML comment on the PR
(`<!-- pr-agent-sha: <sha> -->`), so each run knows exactly where to start.

---

## Prerequisites

1. **Fork / copy** this PR-Agent fork into your own repository or install it as a package.
2. Create an **OpenAI API key** (or whichever LLM provider you configure).
3. Store secrets in your repository's **Settings → Secrets and variables → Actions**:

| Secret name | Value |
|---|---|
| `OPENAI_API_KEY` | Your OpenAI (or provider) API key |
| `GITHUB_TOKEN` | Automatically provided by GitHub — no action needed |

---

## Step 1 — Add the Workflow File

Create `.github/workflows/pr-agent-review.yml` in your **target repository**
(the repo whose PRs you want reviewed):

```yaml
name: PR Agent Review

on:
  pull_request:
    types: [opened, reopened, ready_for_review, synchronize]

# Required so the bot can post comments and read PR data
permissions:
  contents: read
  pull-requests: write
  issues: write

jobs:
  pr-agent-review:
    # Skip draft PRs — remove this line if you want drafts reviewed too
    if: ${{ !github.event.pull_request.draft }}
    runs-on: ubuntu-latest
    timeout-minutes: 10

    steps:
      - name: Run PR-Agent Review
        uses: docker://ghcr.io/unt-dushyant/pr-agent:latest   # or your own image tag
        env:
          OPENAI.KEY: ${{ secrets.OPENAI_API_KEY }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          # --- Full review on open, incremental review on synchronize ---
          GITHUB_ACTION_CONFIG.AUTO_REVIEW: "true"
          # Use incremental flag automatically on push (synchronize) events
          PR_REVIEWER.INCREMENTAL: ${{ github.event.action == 'synchronize' && 'true' || 'false' }}
          # Ensure thresholds don't suppress the review
          PR_REVIEWER.MINIMAL_COMMITS_FOR_INCREMENTAL_REVIEW: "0"
          PR_REVIEWER.MINIMAL_MINUTES_FOR_INCREMENTAL_REVIEW: "0"
```

> **Why not use the GitHub App?** Using a workflow job gives you full control over
> secrets, triggers, and per-repo configuration without running a long-lived server.

---

## Step 2 — Configure via `.pr_agent.toml`

Add a `.pr_agent.toml` file to the **root of your target repository**.
Only include settings you want to override — all other defaults come from
`pr_agent/settings/configuration.toml` in the fork.

```toml
# .pr_agent.toml — place in the root of your target repository

[config]
# Change to your preferred model
model = "gpt-4o"
fallback_models = ["gpt-4o-mini"]

[pr_reviewer]
# Features to enable / disable in every review
require_tests_review          = true
require_security_review       = true
require_estimate_effort_to_review = true
persistent_comment            = true   # updates the same review comment in-place
final_update_message          = false  # quieter; remove or set true for a "review done" ping

# Incremental review thresholds — 0 means every push triggers a review
minimal_commits_for_incremental_review  = 0
minimal_minutes_for_incremental_review  = 0
require_all_thresholds_for_incremental_review = false

[pr_code_suggestions]
commitable_code_suggestions = false   # set true to get one-click apply buttons

[ignore]
glob = [
    # Add your project-specific ignores here
    # e.g. 'src/generated/**'
]
```

---

## Step 3 — (Optional) Build Your Own Docker Image

If you want to pin a specific version or include custom prompts, build from the fork:

```bash
# From the root of this pr-agent fork
docker build -f docker/Dockerfile --target github_action -t ghcr.io/<your-org>/pr-agent:latest .
docker push ghcr.io/<your-org>/pr-agent:latest
```

Then update the `uses:` line in the workflow to point to your image.

---

## Full Review Flow (What Happens)

### On PR Opened
```
pull_request (action: opened)
    │
    ▼
PR_REVIEWER.INCREMENTAL = false
    │
    ▼
get_diff_files()
    Uses repo.compare(merge_base, head_sha)      ← three-way diff
    Fetches full file contents for context
    │
    ▼
AI review of ALL changed files
    │
    ▼
Publishes review comment (persistent)
Stores  <!-- pr-agent-sha: <head_sha> -->  in a hidden comment
```

### On PR Synchronize (New Commit Pushed)
```
pull_request (action: synchronize)
    │
    ▼
PR_REVIEWER.INCREMENTAL = true  → parse_incremental(["-i"])
    │
    ▼
get_reviewed_sha_from_comments()
    Reads  <!-- pr-agent-sha: <prev_sha> -->  from existing PR comments
    │
    ├── prev_sha found?
    │       YES → _get_incremental_commits()
    │                 repo.compare(prev_sha, head_sha)  ← single API call
    │                 Sets unreviewed_files_set to only the changed files
    │
    │       NO  → incremental.is_incremental = False  → full review fallback
    │
    ▼
get_diff_files() — returns only the unreviewed_files_set
    │
    ▼
_can_run_incremental_review() — threshold checks (both 0 → always passes)
    │
    ▼
AI review of ONLY the new diff
    │
    ▼
is_duplicate_comment(pr_review)?
    YES → skip posting (same output already visible)
    NO  → publish new incremental review comment
              Updates  <!-- pr-agent-sha: <new_head_sha> -->
```

---

## Environment Variable Reference

All settings can be overridden via environment variables using the pattern
`SECTION.KEY` (uppercase, dot-separated):

| Env var | `.pr_agent.toml` equivalent | Default |
|---|---|---|
| `CONFIG.MODEL` | `[config] model` | `gpt-4o` |
| `GITHUB_TOKEN` | n/a — always required | — |
| `OPENAI.KEY` | `[openai] key` | — |
| `PR_REVIEWER.INCREMENTAL` | passed as arg `-i` | `false` |
| `PR_REVIEWER.REQUIRE_SECURITY_REVIEW` | `[pr_reviewer] require_security_review` | `true` |
| `PR_REVIEWER.PERSISTENT_COMMENT` | `[pr_reviewer] persistent_comment` | `true` |
| `PR_REVIEWER.MINIMAL_COMMITS_FOR_INCREMENTAL_REVIEW` | `[pr_reviewer] minimal_commits_for_incremental_review` | `0` |

---

## Troubleshooting

### Review runs on every push but shows the full diff
The `<!-- pr-agent-sha: ... -->` tracking comment was not found.  
**Fix:** Make sure the `issues: write` permission is set in the workflow, and that
the first full review successfully completed (it creates the tracking comment).

### "Incremental review is not supported for this provider"
Only `GithubProvider` has `get_incremental_commits`. If you are using GitLab or
Bitbucket, the flag `-i` will be silently ignored and a full review will run.

### Same suggestion appears multiple times
The `is_duplicate_comment` check compares exact text. If the AI rephrases the
suggestion even slightly, it will not be caught.  
**Mitigation:** Set `pr_reviewer.persistent_comment = true` in `.pr_agent.toml`
so the full review is always edited in-place rather than appending new comments.

### Rate limit errors from GitHub API
The incremental review uses a single `repo.compare()` call instead of N per-commit
calls, which greatly reduces API usage. If you still hit limits, increase
`minimal_minutes_for_incremental_review` in `.pr_agent.toml` to add a cooldown
between reviews.
