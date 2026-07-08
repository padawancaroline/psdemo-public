---
name: pipeline-dashboard
description: >
  Generates a rich, interactive HTML dashboard for monitoring data pipelines
  in Treasure Data. Queries workflow execution history, job logs, and audit
  logs to surface KPIs, trend charts, live log streams, an audit trail, and
  a chronological event timeline — all rendered as a polished, self-contained
  HTML file the user can open in any browser.

  Use this skill whenever the user says anything like:
  - "create a pipeline monitoring dashboard"
  - "show me workflow and job status"
  - "visualize my pipeline health"
  - "I want to see job logs and audit logs in a dashboard"
  - "monitor my data pipelines"
  Even if they just say "pipeline dashboard" or "workflow monitoring" with any
  hint of wanting to see or analyze them.
---

# TD Data Pipeline Monitoring Dashboard

## Overview

This skill combines three Treasure Data data sources — workflow execution
history, job logs, and audit logs — into a single monitoring dashboard. It
surfaces active alerts, KPI cards, execution trend charts, filterable tables,
an inline log viewer, an audit trail, and a chronological event timeline.

## Step 1 — Authenticate

```bash
export TDX_ACCESS_TOKEN=$(curl -sf http://172.30.0.1:18080/credentials/td_api_production_aws)
export TDX_SITE=us01
```

If `TDX_ACCESS_TOKEN` is already set and commands succeed, skip this step.

## Step 2 — Gather workflow data

### 2a. List recent workflow sessions (last 24 hours)

```bash
tdx workflow sessions --json --limit 100
```

Key fields to capture per session:

| Field | Purpose |
|-------|---------|
| `id` | Session ID |
| `workflow.name` | Workflow name |
| `project.name` | Project name |
| `status` | `success`, `running`, `error`, `killed` |
| `sessionTime` | Session start timestamp |
| `duration` | Wall-clock duration in seconds |
| `lastAttempt.params` | Trigger source (scheduler, api, etc.) |

### 2b. List workflow definitions

```bash
tdx workflow workflows --json
```

Use this to list all registered workflows with their schedules and projects.

## Step 3 — Gather job logs

### 3a. Recent jobs

```bash
tdx jobs --json --limit 100
```

Key fields per job:

| Field | Purpose |
|-------|---------|
| `id` | Job ID |
| `query` | SQL or task description (truncate at 60 chars for display) |
| `status` | `success`, `running`, `error`, `killed` |
| `num_records` | Rows read |
| `result_size` | Rows written (if available) |
| `cpu_time` | CPU seconds consumed |
| `elapsed_time` | Wall-clock duration in seconds |
| `created_at` | Job creation timestamp |

### 3b. Fetch log for a failed job

For the most recently failed job, fetch its tail log for the inline viewer:

```bash
tdx job log <job_id> --tail 50
```

## Step 4 — Gather audit log data

Run these queries in parallel (they are independent reads):

### 4a. Recent audit events (last 24 hours)
```sql
SELECT
  event_name,
  event_result,
  user_email,
  ip_address,
  resource_type,
  from_unixtime(time) AS event_time
FROM td_audit_log.access
WHERE td_interval(time, '-1d/now')
ORDER BY time DESC
LIMIT 200
```

### 4b. Event type breakdown
```sql
SELECT event_name, COUNT(*) AS cnt
FROM td_audit_log.access
WHERE td_interval(time, '-1d/now')
GROUP BY event_name
ORDER BY cnt DESC
LIMIT 20
```

### 4c. Workflow execution counts by status (last 24 hours)
```sql
SELECT
  COUNT(*) FILTER (WHERE status = 'success') AS succeeded,
  COUNT(*) FILTER (WHERE status = 'error')   AS failed,
  COUNT(*) FILTER (WHERE status = 'running') AS running,
  COUNT(*) FILTER (WHERE status = 'killed')  AS killed
FROM td_workflow_sessions
WHERE td_interval(created_at, '-1d/now')
```

> If `td_workflow_sessions` is not available, derive counts from the
> `tdx workflow sessions` CLI output in Step 2a instead.

### 4d. Hourly job execution volume (last 24 hours)
```sql
SELECT
  DATE_FORMAT(from_unixtime(created_at), '%H') AS hour_utc,
  SUM(CASE WHEN status='success' THEN 1 ELSE 0 END) AS success_cnt,
  SUM(CASE WHEN status='error'   THEN 1 ELSE 0 END) AS error_cnt,
  SUM(CASE WHEN status='running' THEN 1 ELSE 0 END) AS running_cnt
FROM td_jobs
WHERE td_interval(created_at, '-1d/now')
GROUP BY 1
ORDER BY 1
```

> If `td_jobs` system table is unavailable, use the CLI jobs list (Step 3a)
> and bucket by hour client-side.

## Step 5 — Derive KPI numbers

Before generating HTML, compute these values from the collected data:

| KPI | Derivation |
|-----|-----------|
| **Workflows Succeeded** | Count of sessions with `status = success` |
| **Workflows Failed** | Count of sessions with `status = error` or `killed` |
| **Running Now** | Count of sessions with `status = running` |
| **Avg Job Duration** | Mean of `elapsed_time` across all completed jobs |
| **Total Rows Processed** | Sum of `num_records` across all successful jobs |
| **Success Rate (24h)** | `succeeded / (succeeded + failed) × 100`, formatted `XX.X%` |

## Step 6 — Identify active alerts

Scan the data for alert-worthy conditions and surface them at the top:

- **Failed workflow** — any session in `error` state with retries exhausted
- **High queue depth** — pending jobs > 40 (warning threshold)
- **Long-running job** — any job whose `elapsed_time` > 2× the mean
- **Denied audit events** — any `event_result = 'denied'` in last hour

## Step 7 — Generate the HTML dashboard

Write a single self-contained HTML file. All data is embedded as JavaScript
constants — no server required, opens in any browser.

### Required visual sections

1. **Header** — Treasure Data logo mark, dashboard title, live badge, last-refresh countdown, refresh button
2. **Active Alerts strip** — color-coded alert cards (red = error, yellow = warning, blue = info) with time-ago labels
3. **KPI Cards** (6 cards): Succeeded, Failed, Running, Avg Duration, Rows Processed, Success Rate — each with color-coded top border and delta vs. prior day
4. **Trends row** (2 charts side by side):
   - Stacked bar: hourly job executions (Success / Failed / Running) for the last 24 hours
   - Doughnut: job status breakdown with center cutout
5. **Tabbed detail panel** with 4 tabs:
   - **Workflows** — sortable/filterable table with name, project, status badge, duration, progress bar, start time, trigger source
   - **Job Logs** — table with job ID, truncated query, status badge, rows read/written, CPU time, duration, start time; plus inline log viewer showing tail of the most recent failed job
   - **Audit Logs** — filterable list with actor icon, action description (with `<strong>` entity highlights), user, IP, resource, and timestamp; filter buttons for All / Run / Config / Auth / Delete
   - **Timeline** — vertical timeline of step-level events with dot indicators (green = success, red = failed, blue = running), title, detail, and time
6. **Footer** — data sources, generation date

### Design requirements

- Use **Chart.js 4.x** via CDN for all charts
- Dark GitHub-style theme (`#0d1117` bg, `#161b22` surface, `#30363d` border)
- Treasure Data orange accent (`#e85d00` / `#ff7a1f`)
- Status colors: success `#3fb950`, warning `#d29922`, danger `#f85149`, info `#58a6ff`, purple `#bc8cff`, cyan `#39c5cf`
- Status badges with animated dot for "running" state
- Progress bars for workflow completion percentage (orange when < 50%, green otherwise)
- Log viewer in monospace font with per-level color coding: INFO blue, WARN amber, ERROR red, DEBUG muted
- Auto-scroll log viewer to bottom on load
- Live countdown timer in header (counts down to next refresh)
- Fully responsive (single column on mobile)
- Searchable tables with client-side filtering; status filter buttons per tab
- CSV export button per table tab

### Naming the output file

Save as `pipeline_dashboard.html` in the current working directory. After
writing, open the file using `mcp__tas__open_file` so the user sees it immediately.

## Tips

- Truncate SQL queries at 60 characters in the Jobs table; add a `title`
  attribute with the full query for hover display.
- For the timeline, sort events descending by time (most recent at top).
- If `resource_type` is missing from audit data, omit that column silently.
- Color-code audit event icons by type: 🔐 login (cyan), ▶️ run (purple),
  🗑 delete (red), ✏️ update (blue), ➕ create (green), ⚙️ config (amber).
- For the hourly bar chart, cap x-axis labels to every 4 hours to avoid
  crowding on small screens.
- The `"internal"` IP in audit logs dominates totals; if showing an IP
  breakdown, normalize bar widths against the max *external* IP count.
- Round avg duration to the nearest second; display as `Xm Ys` format.

## Related Skills

- **audit-dashboard** — Deeper audit log analysis with 90-day trend charts
- **tdx-basic** — TDX CLI fundamentals, authentication, query syntax
- **segment-analysis** — CDP audience and segment analytics
