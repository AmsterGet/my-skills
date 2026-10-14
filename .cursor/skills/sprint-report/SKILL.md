---
name: sprint-report
description: >-
  Builds an English HTML sprint report for one ReportPortal squad (Jira project
  EPMRPP, scrum board 22585) from live Jira data: scope, goal outcomes, velocity,
  status and focus-area charts, critical defects, and takeaways. Use when the
  user asks for a sprint report or sprint summary and provides a Jira sprint id.
---

# Sprint Report

Produce one HTML sprint report for one squad. Report body is English. The message back to the user is in the conversation language.

Never mix squads. Squad 1 and Squad 2 are different sprints (different id, goal, and people) even when the number matches. If the user asks for both, run this workflow twice, independently, and write two files.

## Inputs

Required in the user message: the Jira sprint id, e.g. `919654`. Squad and sprint number are read from that sprint's name. Do not ask for them, and do not look the sprint up on the board.

If the id is missing, ask before calling Jira. Two squads means two ids and two independent reports.

Optional: output path. Default file name in the workspace root, using `N` and the squad from the sprint name:

- Squad 1 → `sprint-{N}-report.html`
- Squad 2 → `sprint-{N}-squad2-report.html`

## User additions

The user may add their own points, for example an extra row in Sprint Goals & Outcomes, a takeaway, or a comment. Put that text in the report as they wrote it.

Do not call Jira to fill, check, or attach issues to a user-written point unless they explicitly ask. Do not invent a result pill, issue count, or keys for it. If they gave no count, leave the issues cell as `—` and do not fold that row into focus-area totals or KPIs.

## Jira

Tools live in the `user-Jira` MCP namespace. Read each tool schema before the first call.

| Constant | Value |
|---|---|
| Project | `EPMRPP` |
| Board name in the report | Report Portal scrum board |
| Sprint field | `customfield_11200` |
| Page size | `50` (the max). Never stop after one page. |

Do not call `jira_get_sprints_from_board`. Closed sprints are not on the active board, and `state=closed` pages from the oldest sprints (2013), so the squad sprint is not there. Do not use JQL `sprint = "Sprint N"` — that short name mixes squads or matches nothing.

### 1. Resolve the sprint from its id

`jira_search` with `jql=sprint = {id}`, `fields=customfield_11200`, `limit=1`.

If `total` is 0, stop. The id is wrong or the sprint has no issues.

`customfield_11200.value` is a list of GreenHopper strings, one per sprint an issue has visited. Use only the entry whose `id=` equals the requested id. Read fields by their delimiters (`goal` contains commas and newlines, so do not split on every comma):

| Field | Between |
|---|---|
| `goal` | `goal=` and `,id=` |
| `name` | `name=` and `,rapidViewId=` |
| `startDate` | `startDate=` and `,state=` |
| `endDate` | `endDate=` and `,goal=` |

`<null>` means empty. Squad and number come from `name`: it contains `Squad {S}` and `Sprint {N}` not followed by another digit. If it is not Squad 1 or Squad 2, stop and say what the name was.

### Previous sprints for velocity

For each number `N-4` … `N-1`, build the exact name by replacing the number in the resolved name, then `jira_search` with `jql=sprint = "{exact name}"`, `fields=customfield_11200`, `limit=1`. From the returned strings, keep the entry with that name and read its `id`. `total` 0 means that sprint does not exist — omit its bar. Do not guess another name. The report sprint itself is already resolved; its id is the one the user gave.

### 2. Load sprint issues

`jira_get_sprint_issues` with `fields=summary,status,assignee,issuetype,priority,labels` and `limit=50`. Page with `start_at` until the collected count equals `total` from the response. Keep every key, then drop Sub-tasks before any count. The page `total` includes them and is not the report total.

### 3. Velocity

For each of the five sprint ids (skip numbers that do not exist), page `jira_search`:

```
sprint = {id} AND statusCategory = Done AND issuetype != Sub-task
```

`fields=customfield_10004` (Story Points). Sum numeric values; nulls add nothing. If every value is null, repeat with `customfield_35720` (Story Point Estimate) and name that field in the velocity note. Do not switch fields for a single sprint only — one field for the whole chart.

## Counting

**All issues** — every type except `Sub-task` (also drop `Subtask`). Task stays. Each key once. Sub-tasks are excluded from every figure: badge, both KPIs, focus areas, goal-row counts, bus factor, velocity, and the footer total.

**Done** — `status.statusCategory.key` is `done` (or category name `Done`). Anything else is not done.

**Core set** — `issuetype` is exactly `Defect`, `Spike`, or `User Story`.

| Figure | Rule |
|---|---|
| Hero badge | `{total} issues in sprint` |
| KPI 1 | Integer % of **all** issues that are Done. `good` ≥ 90, `warn` ≥ 70, else `bad`. Label: `Closed<br>({done} of {total})`. |
| KPI 2 | Count of **all** issues that are not Done. `good` if 0, else `warn`. If one status name covers them, use it (`In Progress<br>(carried over)`); otherwise `Not done<br>(carried over)`. |
| KPI 3 | `{defects} / {stories} / {spikes}` inside the **core set** only. |
| Status doughnut | Core set only, grouped by status name. Label like `Closed (40)`. Drop empty slices. Colors: Done-category `#4c9a5f`, in-progress category `#3a5bb8`, to-do category `#d4a017`, anything else `#8a94a3`. |
| Focus areas | Partition **all** issues into 5–8 themes aligned with the sprint goal (summary + labels). Counts must sum to the sprint total. Work outside the stated goal is its own area and its goal-table row ends with `(unplanned)`. Say in the status note that the grouping is interpretive and the counts are exact. |
| Goal result | `Done` (pill `done`) when every issue in the row is Done; `In Progress` (pill `progress`) when some are Done; `Open` (pill `open`) when none are Done. |
| Defects table | Core defects with priority `Critical`, `Blocker`, or `Major` only. Sort Blocker, Critical, Major, and open before closed. Priority pill: `crit` for Blocker/Critical, `progress` for Major. Status pill: `done` / `progress` / `open`. If more than 7, show 7 (open and higher priority first) and put the real total in the callout. If zero, omit the table. |
| Bus factor | Per assignee: all-issue count and not-done count. No table. Mention in takeaways only when one person holds a large share. Use Jira display names. `Unassigned` if assignee is null. |

Escape `&`, `<`, `>` in any Jira text placed into HTML.

## HTML

Read `template.html` next to this file. Copy it to the output path. Replace every `{{TOKEN}}`. Do not change CSS, chart types, canvas ids, or layout.

Dates in the hero: `Mon D – Mon D, YYYY` (en-US abbreviations) from `startDate` and `endDate`. If `endDate` is exactly `T00:00:00`, show the previous calendar day. Generated date is today, same style (`Aug 5, 2026`).

`REPORT_NAME` and `REPORT_DESCRIPTION` sit inside JSON. Do not put double quotes or line breaks in them.

| Token | Content |
|---|---|
| `REPORT_NAME` | `Sprint {N} Squad {S} Report` |
| `REPORT_DESCRIPTION` | One sentence: squad, project EPMRPP, what the page contains, and the Jira snapshot date. |
| `PAGE_TITLE` | `Sprint {N} Report — Squad {S}` |
| `HERO_TITLE` | `Sprint {N} — Squad {S}` |
| `HERO_SUB` | `Project EPMRPP · Report Portal scrum board · {start} – {end} · generated {today}` |
| `BADGE` | `{total} issues in sprint` |
| `KPI1_CLASS` / `KPI1_NUM` / `KPI1_LBL` | Class, percent, and label from KPI 1 above. |
| `KPI2_CLASS` / `KPI2_NUM` / `KPI2_LBL` | Class, count, and label from KPI 2 above. |
| `KPI3_NUM` | `{defects} / {stories} / {spikes}` |
| `GOAL_ROWS` | One `<tr>` per stated goal, then unplanned rows. |
| `GOAL_NOTE` | `Sprint goal as stated in Jira: "{exact goal}"`. If unplanned rows exist, say so. If `goal` is empty, say it was empty and themes were inferred from issues. |
| `VELOCITY_NOTE` | Name the SP field and describe the five numbers. Mention missing sprint numbers. |
| `STATUS_NOTE` | Core-set size, plus the focus-area caveat. |
| `DEFECT_BLOCK` | The `<table>` (header Key / Priority / Status / Description) or empty string. |
| `CALLOUT_ICON` | Check mark when no Critical/Blocker remains open; warning sign when one does. |
| `CALLOUT_TEXT` | Whether Critical/Blocker/Major were raised and what is still open. |
| `TAKEAWAY_ITEMS` | 4–6 `<li>`: goal completion, bus factor, defect quality, velocity trend, what stayed open, unplanned load or another real anomaly. |
| `FOOTER_HTML` | See below. |
| `*_JSON` tokens | Raw JSON literals (arrays), not quoted strings. Label count must equal data count. |

Goal row:

```html
<tr>
<td>Goal name</td>
<td><span class="pill done">Done</span></td>
<td>Short comment with example keys.</td>
<td class="key">16</td>
</tr>
```

Defect row:

```html
<tr><td class="key">EPMRPP-1</td><td><span class="pill crit">Critical</span></td><td><span class="pill open">Open</span></td><td>Summary</td></tr>
```

Footer (sprint name is the Jira name, emoji included):

```html
Data source: Jira, sprint <code>{name}</code> (id {id}) · Core set JQL <code>project = EPMRPP AND issuetype in (Defect, Spike, "User Story") AND Sprint = {id}</code> · Total issues: {total} (core set: {core}) · Report generated {date}
```

Every number, key, name, and the goal quote in the Jira-derived parts must come from the Jira responses. A point the user wrote is quoted from them, not looked up. Interpret focus areas and takeaways, but do not invent keys, points, or counts. If a figure is incomplete because a page failed, say so and do not publish a total.

## Before handing the file over

- No `{{` remains.
- Focus-area counts sum to the sprint total.
- Doughnut data sums to the core-set size.
- KPI 1 matches done/total. KPI 3 matches the three core type counts.
- Velocity labels and data have the same length (oldest sprint first).
- Canvas ids `chartVelocity`, `chartStatus`, `chartFocus` are still present.

Then tell the user the file path, sprint id, closed %, carried-over count, and the five velocity numbers. Do not paste the HTML.
