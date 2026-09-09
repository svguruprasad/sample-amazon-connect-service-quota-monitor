# Live Refresh Backend

CloudWatch-powered Lambda + API Gateway that serves real-time metrics to the Connect Operations Dashboard.

## Architecture

```
Dashboard (HTML)  →  API Gateway  →  Lambda  →  CloudWatch + Service Quotas
      ↑                                              ↓
      └────────── JSON response (per request) ───────┘
```

## Deploy

```bash
# Prerequisites: AWS SAM CLI
pip install aws-sam-cli

# Build and deploy
cd live-refresh/
sam build
sam deploy --guided \
    --parameter-overrides \
        ConnectInstanceId=YOUR_INSTANCE_ID
```

SAM prints an `ApiEndpoint` output that ends in `/prod/quota`. The generated
dashboard's 7-day trend chart fetches that URL from the browser, so it needs to
know its own address. To avoid a Function-to-Api circular dependency the address
is passed back in on a second deploy through the `HistoryApiUrl` parameter:

```bash
# First deploy: leave HistoryApiUrl blank, note the ApiEndpoint output.
sam deploy --guided --parameter-overrides ConnectInstanceId=YOUR_INSTANCE_ID

# Second deploy: feed the ApiEndpoint value back in to enable the trend fetch.
sam deploy --parameter-overrides \
    ConnectInstanceId=YOUR_INSTANCE_ID \
    HistoryApiUrl=https://YOUR-API-ID.execute-api.us-east-1.amazonaws.com/prod/quota
```

The static resource report is produced by a separate tool,
`connect-resource-mapper.py` (see the top-level README); it does not consume
this endpoint.

## How It Works

The Lambda runs on a schedule (`ScheduleRate`, default `rate(1 hour)`). On each
run it:

1. Queries CloudWatch and Service Quotas:
   - `AWS/Usage` → per-API call counts (current TPS)
   - `AWS/Connect` → ConcurrentCalls, CallsIncoming (volume)
   - `ServiceQuotas` → current limits (for utilization %)
2. Writes the snapshot to `latest.json` and appends to the `archive/` and
   `peaks/` history in the report bucket.
3. Regenerates `index.html` (the dashboard page) and writes it to the bucket
   root, so the page you open is as fresh as the last scheduled run.

When you open the page, its 7-day trend chart makes a single browser fetch to
`GET /quota?history=7d`. There is no client-side polling loop. A `GET /quota`
with no query string performs a fresh collection on demand.

## API

```
GET /quota                → latest snapshot (fresh collection)
GET /quota?history=1h     → last hour, minute-by-minute (from the archive)
GET /quota?history=1d     → last 24 hours, hourly
GET /quota?history=7d     → last 7 days, daily peaks
GET /quota?history=trend  → last 30 days, daily peaks
```

The endpoint is public read-only (no API key) and rate-limited at the API
Gateway stage. The data is non-sensitive quota-utilization metrics. See the
comment on `DashboardApi` in `template.yaml` if you want to add authentication.

## Configuration

Line config can be provided via:
- `LINE_CONFIG_JSON` env var (inline, for small configs)
- `LINE_CONFIG_S3_BUCKET` + `LINE_CONFIG_S3_KEY` (S3, for larger configs)

## Cost

Costs scale with `ScheduleRate`. At the default `rate(1 hour)` (24 runs/day,
512 MB, ~2s each):

- Lambda: a few cents per month (roughly 720 GB-seconds at 24 runs/day × 512 MB
  × ~2s per run)
- API Gateway: negligible (only browser requests, a handful per dashboard view)
- CloudWatch GetMetricData: the largest line item; each run issues a few
  `get_metric_data` calls (concurrent metrics plus one API-usage query) whose
  total metric count depends on how many APIs and lines you configure, billed
  at $0.01 per 1,000 metrics requested. At the default hourly schedule this is
  a few cents per month.
- **Total: well under $1/month at the default schedule.** A tighter schedule
  (for example `rate(1 minute)`) raises the CloudWatch and Lambda cost roughly
  in proportion to the run count.

## Required IAM Permissions

```yaml
- cloudwatch:GetMetricData
- cloudwatch:GetMetricStatistics
- servicequotas:ListServiceQuotas
- servicequotas:GetServiceQuota
- s3:PutObject   # write latest.json, archive/, peaks/, index.html (report bucket)
- s3:GetObject   # read line config and prior history (report bucket)
```

These are granted by `template.yaml` on the report bucket; the `s3:GetObject`
for line config is only needed if you supply config via S3.
