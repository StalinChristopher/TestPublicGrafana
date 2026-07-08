# CT Code Metrics

This repository is onboarded to [Code & Theory Code Metrics](https://github.com/codeandtheory/ct-github-code-metrics). The integration tracks two complementary signals:

1. **AI attribution** — how much of each commit was written by AI vs a human (Claude Code and Cursor).
2. **Template metrics** — how much of the codebase is still original template code vs custom work since a baseline commit.

Both flows feed a shared DevLake/Grafana dashboard when GitHub Actions secrets are configured.

---

## What is installed in this repo

| Component | Location | Purpose |
|---|---|---|
| Cursor AI hooks | `.cursor/hooks/`, `.cursor/hooks.json` | Snapshot AI output on file edits; install git hook on session stop |
| Claude Code hooks | `.claude/hooks/`, `.claude/settings.json` | Same attribution flow for Claude Code sessions |
| Template provenance | `.template-provenance.json` | Baseline commit and git-diff excludes for template scoring |
| CI workflow | `.github/workflows/template-metrics.yml` | Compute metrics on push/merge and POST to DevLake |
| Cursor setup rules | `.cursor/rules/imported/ct-github-code-metrics/` | Agent rules used during initial integration |

Local snapshot directories (gitignored):

- `.cursor/ai_snapshots/`
- `.claude/ai_snapshots/`

---

## AI attribution

### How it works

1. **On every AI file edit**, a hook saves the exact AI output to a snapshot file under `.cursor/ai_snapshots/` or `.claude/ai_snapshots/`.
2. **When an AI session ends**, a stop hook ensures a `prepare-commit-msg` git hook is installed (without overwriting an existing hook — Claude and Cursor blocks are merged).
3. **On `git commit`**, the prepare-commit-msg hook compares staged content against snapshots and appends an attribution summary to the commit message.

Example stamp in a commit message:

```
─────────────────────────────────────────
🤖 AI Attribution Summary (Cursor)
─────────────────────────────────────────
  Overall  →  AI: 72%  |  Human: 28%
  Generated: 2026-06-29 14:30 UTC

  Per-file breakdown:
    src/components/Header.tsx
      [████████░░]  AI 82% (147 lines)  /  Human 18% (32 lines)
─────────────────────────────────────────
```

If you use both Claude Code and Cursor, you may see separate attribution blocks — one per tool.

### Requirements

- **Git** and **Python 3** on your `PATH`
- Hooks never block commits; failures exit silently

### Hook files

| File | Trigger | Role |
|---|---|---|
| `.cursor/hooks/snapshot_ai_output.py` | Cursor `afterFileEdit` | Save AI-written file content |
| `.cursor/hooks/stop_install_git_hook.py` | Cursor `stop` | Install `prepare-commit-msg` hook |
| `.cursor/hooks/compute_attribution.py` | Git `prepare-commit-msg` | Compute and append attribution |
| `.claude/hooks/snapshot_ai_output.py` | Claude `PostToolUse` (Write/Edit) | Save AI-written file content |
| `.claude/hooks/stop_generate_attribution.py` | Claude `Stop` | Install hook and run attribution preview |

The git hook installs itself automatically the first time an AI session ends after writing files. No manual setup is required beyond committing the hook files in this repo.

---

## Template code metrics

### How it works

On each qualifying CI run, the workflow:

1. Downloads metric scripts from `codeandtheory/ct-github-code-metrics`
2. Computes **template %** and **custom %** by diffing the current tree against a baseline commit
3. Parses **AI attribution %** from git commit message history
4. Uploads `metrics.json` and `ai-attribution.json` as workflow artifacts
5. POSTs combined results to DevLake (when secrets are set)

### Workflow triggers

The **Template Metrics** workflow (`.github/workflows/template-metrics.yml`) runs on:

- Push to `main`
- Pull request closed (merged PRs only)
- Manual `workflow_dispatch`

### Baseline configuration

This repo’s provenance is defined in `.template-provenance.json`:

```json
{
  "schema_version": 1,
  "baseline_commit": "0ea44fef296eae9c5669489eabbc504e6002bc79",
  "extra_excludes": [":!**/*.xcconfig.local", ":!ios/.bundle/**"]
}
```

- **`baseline_commit`** — initial commit from the React Native template (`chore: initial commit from ct-react-native-template`). Lines unchanged since this commit count as template code; everything added or modified counts as custom.
- **`extra_excludes`** — React Native–specific paths excluded from the line count (local Xcode config and Bundler output under `ios/.bundle/`).

To change the baseline (for example after a major template upgrade), update `baseline_commit` to a new SHA or tag and push.

### Optional semantic scoring

The workflow supports Claude-judged **semantic** scoring (`template_pct_semantic`) that ignores cosmetic changes such as Prettier reformats, renames, and comment-only edits. It is **opt-in** and requires the `ANTHROPIC_API_KEY` GitHub Actions secret. When the secret is unset, deterministic git-diff scoring still runs.

---

## GitHub Actions secrets

Configure these in the repository’s GitHub Actions secrets before expecting dashboard data.

| Secret | Required | Purpose |
|---|---|---|
| `CT_METRICS_TOKEN` | Yes | GitHub token to download scripts from `ct-github-code-metrics` |
| `DEVLAKE_WEBHOOK_URL` | For dashboard | DevLake webhook endpoint |
| `DEVLAKE_BASIC_AUTH` | For dashboard | Basic auth for DevLake (format `user:pass`) |
| `ANTHROPIC_API_KEY` | Optional | Enables semantic template scoring |

If `DEVLAKE_WEBHOOK_URL` or `DEVLAKE_BASIC_AUTH` is missing, the workflow still runs and uploads artifacts but skips the DevLake POST.

Set secrets with the GitHub CLI:

```sh
gh secret set CT_METRICS_TOKEN --body "<token>"
gh secret set DEVLAKE_WEBHOOK_URL --body "<url-from-admin>"
gh secret set DEVLAKE_BASIC_AUTH --body "<user:pass>"
gh secret set ANTHROPIC_API_KEY --body "<key>"   # optional
```

Ask your admin for DevLake URL and credentials.

---

## Viewing results

### Locally

- Check commit messages for AI attribution summaries after commits that include AI-edited files.
- Inspect snapshot files under `.cursor/ai_snapshots/` or `.claude/ai_snapshots/` during development (these are not committed).

### In CI

1. Open the **Actions** tab → **Template Metrics** workflow run.
2. Download the **`ct-metrics-<sha>`** artifact containing `metrics.json` and `ai-attribution.json`.

Example fields:

| Metric | Meaning |
|---|---|
| `template_pct` | % of current LOC from the baseline template |
| `custom_pct` | % of current LOC added or changed since baseline |
| `ai_pct` | % of analyzed commit lines attributed to AI |
| `human_pct` | % of analyzed commit lines attributed to humans |

### In Grafana

Once DevLake secrets are configured and the workflow has run on `main`, this repo appears in the shared Grafana dashboard. Subsequent pushes add time-series data points for template, custom, and AI attribution percentages.

---

## Troubleshooting

| Issue | What to check |
|---|---|
| No attribution in commit message | AI session must write files before you commit; snapshots must exist under `.cursor/ai_snapshots/` or `.claude/ai_snapshots/` |
| Git hook not installed | End an AI session after editing files so the stop hook runs; verify `.git/hooks/prepare-commit-msg` exists |
| Workflow fails on script download | Confirm `CT_METRICS_TOKEN` is set and can read `codeandtheory/ct-github-code-metrics` |
| Template metrics show 100% template | Verify `baseline_commit` in `.template-provenance.json` is a valid, reachable SHA |
| DevLake POST skipped | Set both `DEVLAKE_WEBHOOK_URL` and `DEVLAKE_BASIC_AUTH`; check workflow logs for the skip message |
| Python not found | Install Python 3 and ensure `python3` is on `PATH` |

---

## Re-running or updating the integration

Cursor agent rules for re-scaffolding or extending the setup live under:

`.cursor/rules/imported/ct-github-code-metrics/`

Prompt examples:

- “Set up CT metrics”
- “Integrate AI code percentage checker”
- “Set up template metrics for this repo”

Upstream source and full documentation:

**https://github.com/codeandtheory/ct-github-code-metrics**
