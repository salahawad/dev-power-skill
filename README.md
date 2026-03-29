# dev-power-skill

[![Lint](https://github.com/salahawad/dev-power-skill/actions/workflows/lint.yml/badge.svg)](https://github.com/salahawad/dev-power-skill/actions/workflows/lint.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that generates comprehensive developer efficiency reports by cross-referencing **GitLab/GitHub** commit data with **Jira** project management data.

## What it generates

- Executive summary with commit/line/project stats per developer
- Team health dashboard (10 indicators with traffic-light status)
- Quality scorecard (commit consistency, churn, reverts)
- Jira task overview, sprint velocity with trends, completion rates
- Issue health: WIP, blockers, stale items
- Cycle time & lead time from Jira changelog transitions
- Work patterns (peak hours, weekend work, context switching)
- Cross-reference: Jira estimates vs Git output
- MR/PR metrics and review activity
- CI/CD pipeline pass rates per project and developer
- Bus factor analysis per project
- Per-developer breakdown with recent commits
- Actionable recommendations per developer and team-wide

## Installation

### Option 1: Copy into your project

```bash
# From your project root
mkdir -p .claude/commands
cp /path/to/dev-power-skill/.claude/commands/dev-report.md .claude/commands/
```

### Option 2: Clone and symlink

```bash
git clone https://github.com/salahawad/dev-power-skill.git ~/.dev-power-skill
mkdir -p /your/project/.claude/commands
ln -s ~/.dev-power-skill/.claude/commands/dev-report.md /your/project/.claude/commands/dev-report.md
```

## Configuration

Create a `.env` file in your project root (see `.env.example`):

```env
# Git Host (GitLab or GitHub)
GIT_HOST_URL=https://gitlab.example.com/group-name
GITLAB_TOKEN=glpat-xxxxxxxxxxxx

# Jira
JIRA_URL=https://company.atlassian.net
JIRA_PROJECT=DEV
JIRA_EMAIL=user@company.com
JIRA_TOKEN=ATATT3xFfGF0xxxx

# Options
REPORT_PERIOD_DAYS=90
```

### GitLab token

Create a Personal Access Token at `Settings > Access Tokens` with `read_api` scope.

### GitHub token

Create a PAT at `Settings > Developer settings > Personal access tokens` with `repo` and `read:org` scopes.

### Jira token

Create an API token at [id.atlassian.com/manage-profile/security/api-tokens](https://id.atlassian.com/manage-profile/security/api-tokens).

## Usage

In Claude Code, run:

```bash
/dev-report
```

With options:

```bash
/dev-report --period=60 --output=team_report.md
```

The skill will:

1. Read credentials from `.env`
2. Authenticate with both GitLab/GitHub and Jira APIs
3. Collect commits, MRs/PRs, pipelines, and contributor data from Git
4. Collect issues, sprints, changelogs, and story points from Jira
5. Cross-reference and calculate all metrics
6. Generate a full markdown report

## Supported platforms

| Git Host | Jira |
|----------|------|
| GitLab (self-hosted or cloud) | Jira Cloud |
| GitHub (cloud) | |

## License

MIT
