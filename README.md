# dev-power-skill

[![Lint](https://github.com/salahawad/dev-power-skill/actions/workflows/lint.yml/badge.svg)](https://github.com/salahawad/dev-power-skill/actions/workflows/lint.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

## Why this exists

Engineering teams track work across multiple tools — commits in GitLab/GitHub, tasks in Jira — but no single dashboard connects the two. Managers end up manually cross-referencing sprint boards with git logs to answer basic questions:

- Who's overloaded? Who's blocked?
- Are we actually delivering what we planned?
- Which projects have a bus factor of 1?
- Is our review culture healthy or nonexistent?
- How long does work actually take from start to done?

This skill automates that entire process. One command, one report, all the data connected.

## What it is

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) **slash command** (`/dev-report`) that talks directly to the GitLab/GitHub and Jira REST APIs, collects 90 days of activity data, cross-references it, and generates a comprehensive markdown report with actionable findings.

No external dependencies. No SaaS dashboard. No database. Just Claude Code + your API tokens.

## What it generates

A single markdown report (~500-600 lines) covering **23 sections**:

| Category | Sections |
|----------|----------|
| **Overview** | Executive summary, Team health dashboard (traffic-light indicators) |
| **Code quality** | Commit consistency, code churn, reverts, merge noise |
| **Jira delivery** | Task overview, sprint velocity + trends, completion rates, carry-over |
| **Bottlenecks** | WIP overload, blocked issues, stale backlog, issue age |
| **Flow metrics** | Cycle time, lead time (from Jira changelog transitions, not resolution dates) |
| **Work patterns** | Peak hours, weekend work, context switching, inactivity gaps |
| **Cross-reference** | Jira estimates vs actual Git output, commit-to-ticket traceability |
| **Collaboration** | MR/PR metrics, review comments given/received, reviewer coverage |
| **Infrastructure** | CI/CD pipeline pass rates (per project + per developer) |
| **Risk** | Bus factor per project, single-contributor warnings |
| **Per developer** | Individual breakdown with recent commits, project split, cycle time |
| **Recommendations** | Auto-generated findings and actionable items per developer + team-wide |

## Quick start

### 1. Install the skill

```bash
# In your project root
mkdir -p .claude/commands
curl -o .claude/commands/dev-report.md \
  https://raw.githubusercontent.com/salahawad/dev-power-skill/main/.claude/commands/dev-report.md
```

Or clone and symlink:

```bash
git clone https://github.com/salahawad/dev-power-skill.git ~/.dev-power-skill
ln -s ~/.dev-power-skill/.claude/commands/dev-report.md \
  /your/project/.claude/commands/dev-report.md
```

### 2. Add your credentials

Create a `.env` in your project root:

```env
# Git host (GitLab or GitHub — pick one)
GIT_HOST_URL=https://gitlab.example.com/group-name
GITLAB_TOKEN=glpat-xxxxxxxxxxxx
# GITHUB_TOKEN=ghp_xxxxxxxxxxxx

# Jira
JIRA_URL=https://company.atlassian.net
JIRA_PROJECT=DEV
JIRA_EMAIL=user@company.com
JIRA_TOKEN=ATATT3xFfGF0xxxx

# Options
REPORT_PERIOD_DAYS=90
# REPORT_DEVS=Alice,Bob,Charlie   # empty = all devs
```

<details>
<summary>How to get the tokens</summary>

**GitLab** — Create a Personal Access Token at `Settings > Access Tokens` with `read_api` scope.

**GitHub** — Create a PAT at `Settings > Developer settings > Personal access tokens` with `repo` and `read:org` scopes.

**Jira** — Create an API token at [id.atlassian.com/manage-profile/security/api-tokens](https://id.atlassian.com/manage-profile/security/api-tokens). The `JIRA_EMAIL` is the email associated with your Atlassian account.

</details>

### 3. Run it

```bash
/dev-report
```

With options:

```bash
# Specific developers
/dev-report --devs="Alice,Bob,Charlie"

# Custom period and output
/dev-report --period=60 --output=team_report.md

# Combine
/dev-report --devs="Alice,Bob" --period=30
```

If `--devs` is omitted, the skill auto-discovers all developers from the Jira project and Git history, then asks you to confirm the list before generating.

The skill reads `.env`, authenticates with both APIs, collects all data (commits, MRs, pipelines, issues, sprints, changelogs), cross-references it, and writes the report to `developer_efficiency_report.md`.

## How it works

```text
.env credentials
      │
      ├──→ GitLab/GitHub API ──→ commits, MRs/PRs, pipelines, contributors
      │
      └──→ Jira API ──────────→ issues, sprints, changelogs, story points
                                          │
                                    Cross-reference
                                          │
                                          ▼
                               developer_efficiency_report.md
```

Key technical details:

- **Cycle time** is calculated from Jira changelog transitions (In Progress → Done/Test), not `resolutiondate`, which is often null for non-Done statuses
- **Bus factor** counts contributors needed to cover 80% of commits per project
- **Author normalization** groups commits by email to handle developers who commit under different names
- **Sprint carry-over** detects issues appearing in multiple sprints
- All API calls are paginated and rate-limit aware

## Supported platforms

| Git hosting | Project management |
|-------------|-------------------|
| GitLab (self-hosted or cloud) | Jira Cloud |
| GitHub (cloud) | |

## License

MIT
