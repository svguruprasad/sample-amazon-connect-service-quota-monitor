# Amazon Connect Service Quota Monitor and Operations Dashboard

**Predict API quota breaches before they happen — not after your phones go silent.**

---

## What this does

Two tools in one repository:

1. **Quota Monitor** — A Lambda function that checks all 287+ Connect service quotas every hour, alerts on threshold breaches, and writes HTML reports to S3.

2. **Operations Dashboard** — A CLI tool that maps your entire Connect topology (phone numbers, contact flows, Lambdas, quotas) and generates an interactive HTML dashboard with migration wave planning.

---

## Quick start

### Option A: Quota Monitor (deployed Lambda, runs automatically)

```bash
# Deploy the quota monitor Lambda
cd live-refresh/
sam build
sam deploy --guided
```

After deployment, the Lambda runs every hour by default and writes:
- `s3://YOUR_BUCKET/reports/latest.html` (always the current snapshot)
- `s3://YOUR_BUCKET/reports/archive/YYYY-MM-DD/HH-MM.html` (historical)
- `s3://YOUR_BUCKET/reports/peaks/YYYY-MM-DD-peak.json` (daily peak data)

### Option B: Operations Dashboard (CLI, on-demand)

```bash
# Install
pip install boto3

# Run (replace with your instance ID)
python connect-resource-mapper.py \
  --instance-id YOUR_INSTANCE_ID \
  --region us-east-1 \
  --output-dir ./output \
  --line-config line-config.json

# Open results
open ./output/connect-operations-dashboard.html
open ./output/connect-api-report.html
```

---

## How frequently does it run?

| Mode | Frequency | What it produces | Where it writes |
|------|-----------|-----------------|-----------------|
| Quota Monitor Lambda | Every hour (configurable) | Quota utilization snapshot | S3 bucket |
| Live Refresh API | On-demand (API Gateway) | Current metrics for dashboard | API response |
| CLI (resource mapper) | On-demand (you run it) | Full topology + dashboard | Local filesystem |

### Configuring the schedule

The Lambda schedule is set in `live-refresh/template.yaml`:

```yaml
Events:
  ScheduledCheck:
    Type: Schedule
    Properties:
      Schedule: rate(1 hour)   # Change to: rate(15 minutes), rate(4 hours), etc.
      Description: Quota check frequency
```

For migration periods (high risk), set to `rate(15 minutes)`.
For steady state, `rate(1 hour)` or `rate(4 hours)` is sufficient.

---

## How to ensure the same URL always has the latest data

Two approaches:

### Approach 1: S3 static hosting (recommended)

The Lambda writes to a fixed S3 key on every run:

```
s3://YOUR_BUCKET/dashboard/index.html         ← always latest
s3://YOUR_BUCKET/dashboard/api-report.html    ← always latest
```

Enable S3 static website hosting or put CloudFront in front. The URL never changes. Data refreshes every run.

```bash
# S3 static website URL (always latest):
http://YOUR_BUCKET.s3-website-us-east-1.amazonaws.com/dashboard/index.html

# Or behind CloudFront:
https://d1234567.cloudfront.net/dashboard/index.html
```

### Approach 2: API Gateway + embedded fetch

The dashboard HTML includes a `fetch()` call to the Live Refresh API endpoint. When you open the HTML, it pulls fresh data from the API automatically. The HTML file itself is static (cached, CDN-friendly). The data is live.

```
Dashboard HTML (static, cached) → fetches → API Gateway → Lambda → CloudWatch (live)
```

This means you bookmark one URL and it always shows current data.

---

## How archives work

Every run writes to two locations:

```
s3://YOUR_BUCKET/
├── dashboard/
│   ├── index.html                    ← LATEST (overwritten every run)
│   └── api-report.html              ← LATEST (overwritten every run)
├── reports/
│   └── archive/
│       ├── 2026-07-10/
│       │   ├── 08-00.json           ← Hourly snapshot
│       │   ├── 09-00.json
│       │   ├── 10-00.json           ← Peak hour
│       │   └── ...
│       ├── 2026-07-11/
│       │   └── ...
│       └── ...
├── peaks/
│   ├── 2026-07-10-peak.json         ← Daily peak summary
│   ├── 2026-07-11-peak.json
│   └── ...
└── line-config.json                  ← Business line definitions
```

### Archive retention

Set via S3 Lifecycle rule (recommended):
- `reports/archive/` — 90 days retention, then delete
- `peaks/` — 365 days retention (for year-over-year comparison)
- `dashboard/` — no expiration (always latest)

### What's in a peak file?

```json
{
  "date": "2026-07-10",
  "peak_hour": "10:00-11:00 ET",
  "peak_concurrent_calls": 8381,
  "peak_concurrent_chats": 1240,
  "peak_tps": {
    "GetContactAttributes": 47.2,
    "UpdateContactAttributes": 42.8,
    "GetMetricDataV2": 8.1
  },
  "quota_headroom_at_peak": {
    "GetContactAttributes": "21.3%",
    "UpdateContactAttributes": "28.7%",
    "ConcurrentActiveCalls": "58.1%"
  }
}
```

---

## How the dashboard tracks peak hour usage

The dashboard shows three time views:

| View | Data source | What it shows |
|------|-------------|---------------|
| **Today** | CloudWatch GetMetricData (current day) | Hourly bar chart, current TPS, capacity % |
| **This Hour** | CloudWatch (last 60 min, 1-min resolution) | Minute-by-minute TPS, burst detection |
| **7 Days** | Archive folder (daily peak files) | Week trend, peak-hour comparison, day-over-day growth |

### Peak detection logic

Every hourly Lambda run:
1. Pulls `ConcurrentCalls`, `ConcurrentChats`, `ConcurrentTasks` from CloudWatch
2. Pulls per-API TPS from CloudWatch (GetContactAttributes, UpdateContactAttributes, etc.)
3. Compares against quota limits
4. If this hour's values exceed today's previous peak, updates `peaks/YYYY-MM-DD-peak.json`
5. Writes hourly snapshot to `archive/YYYY-MM-DD/HH-MM.json`

The dashboard reads the peaks folder to render the 7-day trend and identify:
- Which hour is consistently the peak (shift change? campaign blast?)
- Whether peak TPS is growing week-over-week
- How much headroom shrinks as migration waves land

---

## Dashboard views explained

### 1. Operations Dashboard (`connect-operations-dashboard.html`)

The main view. Business-line focused.

- **Health strip** — One pill per business line (Claims, Sales, Service, NatGen, Agency). Color = capacity risk.
- **Hourly volume chart** — Bar chart showing call volume by hour. Hover for exact numbers.
- **Capacity meters** — Per-API quota utilization with color coding (green < 70%, yellow 70-85%, red > 85%).
- **Migration wave planner** — Slider: "Add X phone numbers." Shows predicted TPS increase per API and which quota breaches first.
- **Detail panel** — Click any business line to see its specific flows, Lambdas, and per-call API budget.

### 2. API Report (`connect-api-report.html`)

The detail view. Engineering focused.

- **All APIs tab** — Every API called across all flows, sorted by calls-per-contact.
- **Per Flow tab** — Each contact flow with its API breakdown (which APIs, how many times per call).
- **Quotas tab** — Full quota inventory with limit, peak TPS, utilization %, headroom, and daily call count.
- **Lambdas tab** — Every Lambda function with runtime, memory, timeout, Provisioned Concurrency status, and which flows invoke it.

---

## Configuring business lines (`line-config.json`)

The operations dashboard groups phone numbers into business lines for the health strip. Edit `line-config.json` to match your org:

```json
{
  "lines": [
    {
      "id": "claims",
      "name": "Claims",
      "number_pattern": "*-claims-*",
      "tdg_ids": ["8639a7cc-..."],
      "peak_capacity_pct": 78,
      "agent_count": 3000
    },
    {
      "id": "sales",
      "name": "Sales",
      "number_pattern": "*-sales-*",
      "tdg_ids": ["9e209bb8-..."],
      "peak_capacity_pct": 62,
      "agent_count": 1400
    }
  ]
}
```

If you don't provide a line config, the dashboard shows a single "All Traffic" view.

---

## Required IAM permissions

The tool is read-only. It cannot modify any AWS resource.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "connect:List*",
        "connect:Describe*",
        "connect:Get*",
        "lambda:ListFunctions",
        "lambda:GetFunction",
        "lambda:ListProvisionedConcurrencyConfigs",
        "lexv2:ListBots",
        "lexv2:ListBotAliases",
        "servicequotas:ListServiceQuotas",
        "servicequotas:GetServiceQuota",
        "cloudwatch:GetMetricData",
        "cloudwatch:GetMetricStatistics"
      ],
      "Resource": "*"
    }
  ]
}
```

For the Live Refresh Lambda, add:
```json
{
  "Effect": "Allow",
  "Action": ["s3:PutObject", "s3:GetObject"],
  "Resource": "arn:aws:s3:::YOUR_BUCKET/*"
}
```

---

## Deployment options

| Option | Complexity | Best for |
|--------|-----------|----------|
| **CLI only** (run manually, open HTML locally) | Low | One-time assessment, pre-migration planning |
| **Lambda + S3** (automated, static hosting) | Medium | Ongoing monitoring, team-shared dashboard |
| **Lambda + API Gateway + CloudFront** (live data) | Medium-High | Real-time ops center, NOC screen |

### Deploying Lambda + S3 (recommended for ongoing use)

```bash
cd live-refresh/
sam build
sam deploy \
  --stack-name connect-ops-dashboard \
  --parameter-overrides \
    ConnectInstanceId=YOUR_INSTANCE_ID \
    LineConfigJson='{"lines": [...]}' \
  --capabilities CAPABILITY_IAM
```

After deployment, the output shows the API Gateway URL. Add it to the dashboard HTML or access it directly.

---

## Accessibility

The dashboard HTML complies with WCAG 2.1 AA:

- Color indicators paired with shape icons (circle = green, triangle = yellow, square = red)
- All interactive elements have aria-labels
- Tab navigation works with keyboard (Tab, Enter, Arrow keys)
- Font sizing in rem (respects browser zoom preference)
- Print stylesheet included
- Tested with VoiceOver (macOS) and NVDA (Windows)
- `prefers-reduced-motion` media query disables animations
- Colorblind-safe palette option (pass `--colorblind` to mapper or set `"colorblind": true` in line-config.json)

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| All quotas show 0% utilization | Idle instance (no traffic) or CloudWatch not publishing | Run during business hours. Check that Connect instance has active calls. |
| ConcurrentCalls shows 0 | MetricGroup dimension missing | Fixed in this version. Ensure you're running latest code. |
| Scan takes > 5 minutes | Large instance (50K+ numbers) | Normal. Pagination adds time. |
| "Access Denied" errors | IAM policy missing permissions | Apply the IAM policy above. Check the specific API in the error. |
| Dashboard shows "No data" | Output JSON missing or empty | Re-run the mapper. Check that output-dir exists. |
| Live refresh returns stale data | Lambda schedule too infrequent | Reduce interval: `rate(15 minutes)` during migrations. |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  CLI: connect-resource-mapper.py                                │
│                                                                 │
│  Phone Numbers → TDGs → Contact Flows → Lambdas → APIs → Quotas│
│       (45K)       (4)       (200+)        (160)    (20+)  (287) │
│                                                                 │
│  Outputs:                                                       │
│    connect-resource-map.json      (topology graph)              │
│    connect-quota-impact-model.json (predictive model)           │
│    connect-operations-dashboard.html                            │
│    connect-api-report.html                                      │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  Lambda: live-refresh                                           │
│                                                                 │
│  EventBridge (hourly) → Lambda → CloudWatch + ServiceQuotas     │
│                             │                                   │
│                             ├→ S3: dashboard/index.html (latest)│
│                             ├→ S3: archive/YYYY-MM-DD/HH.json   │
│                             └→ S3: peaks/YYYY-MM-DD-peak.json   │
│                                                                 │
│  API Gateway → Lambda → Live metrics JSON response              │
│  (dashboard fetches this on page load for real-time view)       │
└─────────────────────────────────────────────────────────────────┘
```

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## License

MIT-0 (No attribution required). See [LICENSE](LICENSE).
