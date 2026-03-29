---
description: Generate a developer efficiency report from GitLab/GitHub + Jira data. Usage: /dev-report [--period=90] [--output=developer_efficiency_report.md]
---

# Developer Efficiency Report Generator

You are generating a comprehensive developer efficiency report by cross-referencing Git hosting (GitLab or GitHub) data with Jira project data.

## 1. Configuration & Authentication

### Read configuration

First, check for a `.env` file in the project root. Extract these variables:

| Variable | Required | Description |
|----------|----------|-------------|
| `GITLAB_TOKEN` | If using GitLab | GitLab Personal Access Token (`glpat-...`) |
| `GITHUB_TOKEN` | If using GitHub | GitHub PAT (`ghp_...` or `github_pat_...`) |
| `JIRA_TOKEN` | Yes | Jira API Token (`ATATT3x...`) |
| `JIRA_EMAIL` | Yes | Email for Jira basic auth |
| `GIT_HOST_URL` | Yes | e.g. `https://gitlab-ce.hapster.dev/hapster-product` or `https://github.com/org-name` |
| `JIRA_URL` | Yes | e.g. `https://company.atlassian.net` |
| `JIRA_PROJECT` | Yes | Jira project key, e.g. `DEV` |
| `REPORT_PERIOD_DAYS` | No | Default: 90 |

If any variable is missing from `.env`, check `$ARGUMENTS` for overrides (e.g., `--period=90`). If still missing, **ask the user** for the required values before proceeding.

### Detect platform

- If `GIT_HOST_URL` contains `gitlab` → use GitLab API v4
- If `GIT_HOST_URL` contains `github` → use GitHub REST API v3
- Otherwise, ask the user

### Verify authentication

Run these in parallel:

**GitLab:**

```bash
curl -s -o /dev/null -w "%{http_code}" -H "PRIVATE-TOKEN: $GITLAB_TOKEN" "$GITLAB_API/groups?search=GROUP_NAME"

```

**GitHub:**

```bash
curl -s -o /dev/null -w "%{http_code}" -H "Authorization: Bearer $GITHUB_TOKEN" "https://api.github.com/orgs/ORG_NAME"

```

**Jira:**

```bash
curl -s -o /dev/null -w "%{http_code}" -u "$JIRA_EMAIL:$JIRA_TOKEN" "$JIRA_URL/rest/api/3/myself"

```

If any return non-200, report the error and stop. For Jira, if `JIRA_EMAIL` is not set, try the git config email: `git config user.email`.

## 2. Data Collection — Git Host

Set `SINCE_DATE` based on period (default 90 days). Use ISO format: `YYYY-MM-DD`.

### 2.1 Discover projects

**GitLab:** Extract group path from URL, then:

```bash

# Get group ID first
curl -s -H "PRIVATE-TOKEN: $TOKEN" "$API/groups?search=$GROUP_PATH" | python3 -c "import sys,json; [print(g['id']) for g in json.load(sys.stdin) if g['path']=='$GROUP_PATH']"

# Get all projects in group (paginate)
curl -s -H "PRIVATE-TOKEN: $TOKEN" "$API/groups/$GID/projects?per_page=100&include_subgroups=true&with_shared=false"

```

**GitHub:**

```bash
curl -s -H "Authorization: Bearer $TOKEN" "https://api.github.com/orgs/$ORG/repos?per_page=100&sort=pushed&type=all"

```

Store project IDs and names. Filter to projects with activity after `SINCE_DATE`.

### 2.2 Collect commits per project

For each active project, paginate through all commits since `SINCE_DATE`:

**GitLab:**

```bash
curl -s -H "PRIVATE-TOKEN: $TOKEN" "$API/projects/$PID/repository/commits?since=$SINCE_DATE&per_page=100&page=$PAGE"

```

**GitHub:**

```bash
curl -s -H "Authorization: Bearer $TOKEN" "https://api.github.com/repos/$OWNER/$REPO/commits?since=${SINCE_DATE}T00:00:00Z&per_page=100&page=$PAGE"

```

For each commit, extract:

- `author_name` / `author_email`
- `committed_date` / `created_at` (timestamp with timezone)
- `title` / `message`
- `stats.additions`, `stats.deletions` (GitLab: available per commit; GitHub: need individual commit endpoint)

**Important:** For GitHub, the list endpoint doesn't include stats. Fetch individual commits for line counts:

```bash
curl -s -H "Authorization: Bearer $TOKEN" "https://api.github.com/repos/$OWNER/$REPO/commits/$SHA"

```

### 2.3 Classify commits

For each commit, classify by message pattern:

- **Merge:** message starts with `Merge branch` or `Merge pull request`
- **Revert:** message starts with `Revert`
- **Fix:** message contains `fix` (case-insensitive) or starts with `fix:`, `fix(`
- **Feature:** everything else (default)

### 2.4 Collect merge request / pull request data

**GitLab:**

```bash
curl -s -H "PRIVATE-TOKEN: $TOKEN" "$API/projects/$PID/merge_requests?state=all&created_after=$SINCE_DATE&per_page=100"

```

For each MR, also fetch notes (comments):

```bash
curl -s -H "PRIVATE-TOKEN: $TOKEN" "$API/projects/$PID/merge_requests/$MR_IID/notes?per_page=100"

```

**GitHub:**

```bash
curl -s -H "Authorization: Bearer $TOKEN" "https://api.github.com/repos/$OWNER/$REPO/pulls?state=all&sort=created&direction=desc&per_page=100"

```

For each PR, fetch reviews:

```bash
curl -s -H "Authorization: Bearer $TOKEN" "https://api.github.com/repos/$OWNER/$REPO/pulls/$PR_NUMBER/reviews"

```

Extract per MR/PR:

- Author, state (opened/merged/closed), created_at, merged_at, closed_at
- Review comments count
- Reviewers

### 2.5 Collect pipeline / workflow data

**GitLab:**

```bash
curl -s -H "PRIVATE-TOKEN: $TOKEN" "$API/projects/$PID/pipelines?updated_after=$SINCE_DATE&per_page=100"

```

Each pipeline has `status` (success/failed/canceled), `ref`, and `user.username`.

**GitHub:**

```bash
curl -s -H "Authorization: Bearer $TOKEN" "https://api.github.com/repos/$OWNER/$REPO/actions/runs?created=>$SINCE_DATE&per_page=100"
```

Each run has `conclusion` (success/failure/cancelled), `head_branch`, `actor.login`.

### 2.6 Collect contributor data (for bus factor)

**GitLab:**

```bash
curl -s -H "PRIVATE-TOKEN: $TOKEN" "$API/projects/$PID/repository/contributors?per_page=100"

```

**GitHub:**

```bash
curl -s -H "Authorization: Bearer $TOKEN" "https://api.github.com/repos/$OWNER/$REPO/contributors?per_page=100"

```

## 3. Data Collection -- Jira

### 3.1 Get team members

First, discover developers from the Jira project:

```bash
curl -s -u "$EMAIL:$TOKEN" "$JIRA_URL/rest/api/3/search?jql=project=$PROJECT AND updated >= -${PERIOD}d&fields=assignee&maxResults=0"

```

Then get unique assignees:

```bash
curl -s -u "$EMAIL:$TOKEN" "$JIRA_URL/rest/api/3/search?jql=project=$PROJECT AND assignee is not EMPTY AND updated >= -${PERIOD}d&fields=assignee&maxResults=100"

```

Build a unique list of developer display names + account IDs.

### 3.2 Get all issues with changelogs

This is the main Jira query. Paginate through ALL issues:

```bash
curl -s -u "$EMAIL:$TOKEN" "$JIRA_URL/rest/api/3/search?jql=project=$PROJECT AND updated >= -${PERIOD}d&fields=assignee,created,updated,resolutiondate,status,issuetype,priority,sprint,story_points,customfield_10016&expand=changelog&maxResults=100&startAt=$OFFSET"

```

**Note:** Story points may be in `customfield_10016` (standard) or another custom field. Check the first response to identify the correct field. Common names: `story_points`, `Story Points`, `customfield_10016`, `customfield_10028`.

For each issue, extract:

- Key (e.g., DEV-1234)
- Summary
- Assignee display name
- Status name and category
- Issue type (Bug, Story, Task, Sub-task)
- Story points
- Sprint name(s) — an issue can appear in multiple sprints (carry-over)
- Created date, updated date, resolution date
- Changelog: all status transitions with timestamps

### 3.3 Get sprint data

```bash
curl -s -u "$EMAIL:$TOKEN" "$JIRA_URL/rest/agile/1.0/board?projectKeyOrId=$PROJECT"

```

Get the board ID, then:

```bash
curl -s -u "$EMAIL:$TOKEN" "$JIRA_URL/rest/agile/1.0/board/$BOARD_ID/sprint?state=active,closed&maxResults=50"

```

For each sprint, extract: name, start date, end date, state.

## 4. Data Processing

Use Python (via bash `python3 -c` or a temp script) for all calculations. Process ALL the collected data into the following metrics:

### 4.1 Per-Developer Git Metrics

For each developer (group by normalized author name):

- **Total commits**, **features**, **fixes**, **merges**, **reverts** (from classification)
- **Lines added**, **lines deleted** (sum of stats)
- **Commit size stats**: median, mean, max, coefficient of variation (CV = std/mean)
- **Active days**: unique dates with commits
- **Commit frequency**: total commits / number of weeks in period
- **Effective commit frequency**: (total - merges) / weeks
- **Active date range**: first commit date to last commit date
- **Projects**: unique project list and commit count per project
- **Work pattern analysis**:
  - Extract hour from each commit timestamp (convert to local timezone)
  - Bucket into: Morning (7-12), Afternoon (12-17), Evening (17-22), Late night (22-7)
  - Find peak hours (top 3)
  - Count weekend commits (Saturday/Sunday)
  - Find busiest day of week

- **Context switching**: count days where developer committed to 2+ projects
- **Max inactivity gap**: largest gap between consecutive active days
- **Code churn estimate**: min(lines_added, lines_deleted) — approximation of rework
- **Recent commits**: last 15 non-merge commits with date, project, type, message, +/-

### 4.2 Per-Developer Jira Metrics

For each developer:

- **Total issues** assigned
- **Story points** total (sum across all issues)
- **Status distribution**: count issues in each status
- **Issue type distribution**: count by Bug, Story, Task, Sub-task
- **Completed issues**: status in (Done, Test) or status category = Done
- **Dropped issues**: status = Dropped/Cancelled/Won't Do
- **Bugs**: count of Bug-type issues
- **SP estimation coverage**: % of issues that have story points assigned
- **WIP count**: issues in active statuses (In Progress, Code Review, Test, Blocked, etc.) — NOT Done, NOT To Do
- **Backlog**: issues in To Do status
- **Blocked / Needs info**: count of issues in blocking statuses
- **Stale To Do**: issues in To Do status older than 14 days
- **Issues > 30 days old**: open issues created more than 30 days ago
- **SP by status**: sum of story points grouped by status

### 4.3 Cycle Time & Lead Time (from Jira changelogs)

For each issue with changelog data:

1. Find the FIRST transition TO "In Progress" (or similar active status)
2. Find the FIRST transition TO "Done" or "Test" (or similar completed status)
3. **Cycle time** = (step 2 timestamp) - (step 1 timestamp), in days
4. **Lead time** = (step 2 timestamp) - (issue created date), in days

Per developer, calculate:

- Average, median, P75, P90, max cycle time
- Average, median, max lead time

### 4.4 Sprint Velocity

For each sprint, for each developer:

- Count assigned issues
- Count completed (Done + Test)
- Count remaining (To Do)
- Count dropped
- Sum story points completed
- Calculate completion % = (Done+Test) / Assigned * 100

**Sprint carry-over rate** per developer: count issues that appear in 2+ sprints / total issues.

### 4.5 Cross-Reference: Jira vs Git

- **Commits per Jira issue** = non-merge commits / total Jira issues
- **Lines of code per SP** = total lines added / total story points
- **Commit-to-ticket traceability**: count commits whose message contains a Jira ticket pattern (e.g., `DEV-\d+`), divided by total non-merge commits

### 4.6 Merge Request Metrics

Per developer:

- MRs opened, merged, closed (rejected)
- Merge rate = merged / opened * 100
- Average turnaround time = avg(merged_at - created_at) or avg(closed_at - created_at)
- Review comments received (total notes/comments on their MRs)
- Review comments given (comments they left on others' MRs)
- Unique MRs reviewed

### 4.7 Pipeline Health

Per project:

- Total pipelines, passed, failed, canceled
- Pass rate = passed / total * 100

Per developer (from pipeline user field):

- Their pipeline count, passed, failed, pass rate

### 4.8 Bus Factor

Per project (from contributor data, filtered to period):

- Count contributors with >5% of commits
- Identify primary contributor and their ownership %
- Bus factor = number of contributors needed to cover 80% of commits

### 4.9 Team Health Summary

Evaluate each indicator as green/yellow/red:

| Indicator | Green | Yellow | Red |
|-----------|-------|--------|-----|
| Velocity trend | Increasing or stable | Flat | Decreasing |
| Code review culture | >50% MRs reviewed | 20-50% | <20% |
| Commit traceability | >50% | 20-50% | <20% |
| WIP discipline | All devs <10 active | Some >10 | Most >15 |
| CI/CD coverage | All projects have CI | >50% have CI | <50% have CI |
| Bus factor | All >2 | Some =1 | Most =1 |
| Cycle time | Median <3d | 3-10d | >10d |
| Sprint carry-over | <10% | 10-25% | >25% |
| Blocked items | 0 | 1-3 | >3 |
| Backlog hygiene | <5 stale items | 5-10 | >10 |

## 5. Report Output Format

Generate the report as a markdown file. Use the exact structure below. Replace all `{{placeholders}}` with computed data.

```markdown

# Developer Efficiency Report

**Generated:** {{YYYY-MM-DD HH:MM}} UTC
**{{Platform}}:** [{{group/org name}}]({{git_host_url}})
**Period:** Last {{period}} days (since {{since_date}})
**Projects analyzed:** {{project_count}}

---

## Executive Summary

| Developer | Commits | Lines Added | Lines Deleted | Active Days | Projects | Fix % | Revert |
|-----------|---------|-------------|---------------|-------------|----------|-------|--------|
{{for each developer, one row}}

---

## Team Health Summary

| Indicator | Status | Detail |
|-----------|--------|--------|
| Velocity trend | {{:green_circle:/:orange_circle:/:red_circle:}} | {{detail}} |
| Code review culture | {{color}} | {{detail}} |
| Commit traceability | {{color}} | {{detail}} |
| WIP discipline | {{color}} | {{detail}} |
| CI/CD coverage | {{color}} | {{detail}} |
| Bus factor | {{color}} | {{detail}} |
| Cycle time | {{color}} | {{detail}} |
| Sprint carry-over | {{color}} | {{detail}} |
| Blocked items | {{color}} | {{detail}} |
| Backlog hygiene | {{color}} | {{detail}} |

---

## Quality Scorecard

| Metric | {{dev1}} | {{dev2}} | ... |
|--------|---------|---------|-----|
| Commit size consistency | {{Consistent/Erratic}} (CV={{cv}}) | ... |
| Bug-fix ratio | {{fix_pct}}% ({{low/moderate/high}}) | ... |
| Reverts | {{count}} {{warn if >0}} | ... |
| Est. code churn | {{churn}} lines ({{churn_pct}}% {{warn if >30%}}) | ... |
| Merge commit noise | {{count}} ({{pct}}%) | ... |

> **Code churn** = lines deleted that overlap with lines added in the same period. High churn % indicates rework.

---

## Jira -- Task Overview

**Project:** [{{project_key}}]({{jira_url}}/jira/software/c/projects/{{project_key}}/boards/{{board_id}})

| Developer | Issues | Story Points | Completed (Done+Test) | Dropped | Bugs | SP Estimation Coverage |
|-----------|--------|-------------|----------------------|---------|------|----------------------|
{{rows}}

### Status Distribution
{{status count table, developers as columns}}

### Issue Type Distribution
{{type count table}}

---

## Jira -- Sprint Velocity

### Story Points per Sprint

| Sprint | {{dev columns}} | Team Total | Trend |
{{rows with trend arrows: up/down/stable}}

### Sprint Completion Breakdown (issues)

| Sprint | Dev | Assigned | Done+Test | To Do | Dropped | Completion % | SP Completed |
{{rows, flag completion <80% with warning}}

---

## Jira -- Issue Health & Bottlenecks

| Metric | {{dev columns}} |
|--------|{{cols}}|
| Current WIP (active issues) | {{count, warn if >15}} |
| Backlog (To Do) | {{count}} |
| Blocked / Needs info | {{count, bold if >0}} |
| Avg cycle time (In Progress->Done/Test) | {{days}} |
| Median cycle time | {{days}} |
| Avg lead time (Created->Done/Test) | {{days}} |
| Sprint carry-over rate | {{pct, warn if >25%}} |
| Stale To Do (>14 days) | {{count, warn if >5}} |
| Issues > 30 days old (open) | {{count}} |

### Story Points by Status
{{SP sum per status per developer}}

### Blocked Issues (action needed)
| Developer | Key | Status | Age | SP | Summary |
{{rows for all blocked/needs-info issues}}

### Stale To Do Items (>14 days, should be re-prioritized or dropped)
| Developer | Key | Age | SP | Summary |
{{rows}}

---

## Work Patterns (commit timestamps)

| Metric | {{dev columns}} |
| Peak hours | {{top 3 hours}} |
| Morning (7-12) | {{pct}}% |
| Afternoon (12-17) | {{pct}}% |
| Evening (17-22) | {{pct}}% |
| Late night (22-07) | {{pct}}% |
| Weekend commits | {{count}} ({{pct}}%) |
| Busiest day | {{day}} ({{count}}) |

### Focus & Context Switching

| Metric | {{dev columns}} |
| Multi-project days | {{count}}/{{active_days}} ({{pct}}%) |
| Max inactivity gap | {{days}} days |
| Avg gap between active days | {{days}} days |
| Jira WIP count | {{count}} {{warn if >15}} |

---

## Cross-Reference: Jira vs GitLab/GitHub

| Metric | {{dev columns}} |
| Jira issues | {{count}} |
| Commits (non-merge) | {{count}} |
| Commits per Jira issue | {{ratio}} |
| Story points total | {{sp}} |
| Lines of code per SP | {{loc_per_sp}} |
| Dropped/Total issues | {{dropped}}/{{total}} ({{pct}}%) |
| Avg SP per issue | {{avg}} |

> LOC/SP is a rough signal, not a productivity measure. Bulk refactors inflate this metric.

### Commit-to-Ticket Traceability

| Developer | Commits w/ ticket ref | Total non-merge | Traceability |
{{rows, warn if <20%}}

---

## Merge Request Metrics

| Metric | {{dev columns}} |
| MRs opened | |
| MRs merged | |
| MRs closed (rejected) | |
| Merge rate | |
| Avg MR turnaround | |
| Review comments received | |

---

## Review & Collaboration

| Metric | {{dev columns}} |
| Review comments given | |
| Unique MRs reviewed | |
| Cross-project breadth | |

---

## Commit Classification

| Developer | Features | Fixes | Merges | Reverts | Total |
{{rows}}

---

## Per-Developer Breakdown

{{For each developer:}}

### {{Developer Name}}

- **Commit frequency:** ~{{freq}} commits/week (effective: ~{{eff_freq}}/week excl. merges)
- **Active date range:** {{first}} to {{last}}
- **Active days:** {{count}}
- **Total commits:** {{total}} (features: {{f}}, fixes: {{x}}, merges: {{m}})
- **Cycle time:** avg {{avg}}d, median {{med}}d | **Lead time:** avg {{lead}}d
- **Lines added:** +{{added}}
- **Lines deleted:** -{{deleted}}
- **Commit size:** median={{med}}, mean={{mean}}, max={{max}}

| Project | Commits | Added | Deleted |
{{project breakdown rows}}

<details>
<summary>Recent commits (15 shown)</summary>

| Date | Project | Type | Message | +/- |
{{last 15 non-merge commits}}

</details>

---

## Weekly Activity Comparison

| Week | {{dev columns}} |
{{rows: ISO week number, commit count per dev}}

---

## Cycle Time & Lead Time

| Metric | {{dev columns}} |
| Completed issues | |
| Avg cycle time | |
| Median cycle time | |
| P90 cycle time | |
| Avg lead time | |
| Median lead time | |
| Sprint carry-over rate | |

> Cycle time = first "In Progress" -> first "Done/Test" from Jira changelog transitions.
> Lead time = issue created -> first "Done/Test".

---

## CI/CD Pipeline Health

### By Project

| Project | Pipelines | Passed | Failed | Canceled | Pass Rate |
{{rows, "No CI" for projects without pipelines}}

### By Developer

| Developer | Project | Pipelines | Passed | Failed | Pass Rate |
{{rows}}

---

## Bus Factor

| Project | Bus Factor | Primary Contributor | Ownership % | Other Contributors |
{{rows, warn bus factor = 1}}

---

## Key Findings & Recommendations

{{For each developer, generate:}}

### {{Developer Name}}

**Issues identified:**
{{Bullet list of issues. Draw from ALL metrics above. Flag:}}

- Bug-fix ratio >20% as high
- Any reverts
- CV >2.0 as erratic commit sizes
- Zero review comments given
- Traceability <20%
- WIP >15
- Stale To Do >5
- Open issues >30 days old count
- Code churn >40%
- Bus factor =1 on their primary projects
- No CI on their primary projects
- Sprint completion <80%
- Sprint carry-over >20%
- High merge commit noise >30%
- MR merge rate <80%
- Context switching >50% of days
- Lead time outliers

**Recommendations:**
{{Numbered list, actionable, specific. Always include:}}
1. WIP reduction if applicable
2. Backlog grooming if stale items
3. Code review participation
4. Traceability improvement
5. Bus factor mitigation
6. CI setup if missing
7. Any developer-specific issues

### Team-wide observations
{{Bullet list of cross-cutting concerns}}

---
*Report generated automatically from {{platform}} and Jira API data.*

```

## 6. Output

Write the report to the file specified in `$ARGUMENTS` (default: `developer_efficiency_report.md` in the project root).

## 7. Implementation Notes

- **Pagination**: Always paginate API responses. GitLab uses `page` param (check `x-total-pages` header). GitHub uses `Link` header. Jira uses `startAt` + `maxResults`.
- **Rate limits**: GitHub has 5000 req/hr for authenticated requests. Add 0.1s delay between batch calls if needed. GitLab self-hosted typically has no rate limit. Jira Cloud allows ~100 req/min.
- **Author normalization**: Developers may commit under different names/emails. Group by common patterns (e.g., `michael.d` and `Michael Debs` are the same person). Use email as the primary key when available.
- **Timezone**: Convert all commit timestamps to a consistent timezone (use the system timezone or UTC).
- **Large datasets**: For repos with >1000 commits in the period, use `python3` scripts written to temp files rather than inline `-c` commands.
- **Error handling**: If an API call fails, log the error and continue with available data. Note any data gaps in the report footer.
- **Story points field**: Jira story points may be in different custom fields. Try `customfield_10016` first, then search for a field named "Story Points" or "Story point estimate" via `GET /rest/api/3/field`.

## 8. Quick Start

If the user just runs `/dev-report` with no arguments, check `.env` for all required variables. If `.env` has `GITLAB_TOKEN`/`GITHUB_TOKEN` and `JIRA_TOKEN`, auto-detect the platform and run. Otherwise prompt for missing config.

Typical `.env` setup:

```env
GIT_HOST_URL=https://gitlab-ce.example.com/group-name
GITLAB_TOKEN=glpat-xxxxxxxxxxxx
JIRA_URL=https://company.atlassian.net
JIRA_PROJECT=DEV
JIRA_EMAIL=user@company.com
JIRA_TOKEN=ATATT3xFfGF0xxxx
REPORT_PERIOD_DAYS=90

```
