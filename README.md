# 📊 ERP Server — API Observability Dashboard (Grafana + Loki)

> A production-grade Grafana dashboard for real-time observability of an **ASP.NET Core ERP API**, built on top of **Serilog structured logs** shipped to **Grafana Loki**.

![Grafana](https://img.shields.io/badge/Grafana-F46800?style=for-the-badge&logo=grafana&logoColor=white)
![Loki](https://img.shields.io/badge/Loki-F0AC35?style=for-the-badge&logo=grafana&logoColor=white)
![ASP.NET Core](https://img.shields.io/badge/ASP.NET_Core-512BD4?style=for-the-badge&logo=dotnet&logoColor=white)
![Serilog](https://img.shields.io/badge/Serilog-C94634?style=for-the-badge&logo=dotnet&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)

---

## 📌 Overview

This dashboard was built to solve a real problem: having **no visibility** into how an ERP API was performing in production. Using only Loki log streams (no Prometheus/metrics), it extracts meaningful observability signals directly from structured JSON logs written by Serilog.

The result is a fully functional monitoring dashboard that shows request volume, error rates, slow endpoints, and live log streams — all from a single log datasource.

---

## 🧱 Stack

| Layer | Technology |
|---|---|
| API | ASP.NET Core (ERP backend) |
| Logging | Serilog → JSON structured logs |
| Log Aggregation | Grafana Loki |
| Visualization | Grafana 12 |
| Infrastructure | VPS managed via Dokploy + Traefik |
| Containerization | Docker |

---

## ✨ Dashboard Features

### 📊 Overview Row
| Panel | Description |
|---|---|
| 🚀 Total Requests | Count of all HTTP requests in the selected time range |
| 🔴 Total Errors (4xx + 5xx) | Error count with yellow/red thresholds |
| ✅ Total 2xx Successes | Successful request count |
| 🟡 400 Bad Requests | Validation/bad input errors |
| 🔐 401 Unauthorized | Auth failures — key security signal |
| 🐌 Slow Requests > 2s | Count of requests exceeding 2 second threshold |

### 📊 Status Code Distribution
- **Donut pie chart** — visual breakdown of 200 / 400 / 401 / 404 / 5xx share
- **Horizontal bar gauge** — side-by-side count comparison per status code
- **🔎 Filtered Log Viewer** — use the **Status Code** dropdown to instantly filter and read the actual log entries for any status class

### 🐌 Slow API Tracking
- **Stat card** — total count of slow requests (>2s) — turns red when elevated
- **Endpoint detail table** — lists every slow endpoint with:
  - Avg latency (ms) — gradient gauge
  - Max latency (ms) — worst-case hit
  - Hit count — how often it breached threshold
- **Full detail log viewer** — raw prettified JSON log entries for every slow request

### 🛣️ Endpoint Intelligence
- **Top endpoints by request count** — table with inline bar gauge
- **Slowest endpoints by avg latency** — table with gradient gauge, sorted descending

### 📈 Errors Over Time
- Stacked bar chart: **4xx / 5xx / 401 Auth Failures** over time
- Log volume by service: **ERPServer.API** vs **auth** side by side

### 🔍 Log Streams
- **Error logs full detail** — prettified, expandable JSON for all 4xx/5xx entries
- **All logs live stream** — full real-time log tail with label display

---

## 🔧 How It Works

All panels are powered by **LogQL** — Loki's query language — extracting metrics directly from log lines. No Prometheus scraping needed.

### Key LogQL techniques used

**Counting log lines matching a filter:**
```logql
sum(count_over_time(
  {service_name=~"$service", SourceContext="Serilog.AspNetCore.RequestLoggingMiddleware"}
  | json | StatusCode >= 400 [$__range]
))
```

**Unwrapping a numeric field to compute latency averages:**
```logql
avg_over_time(
  {service_name=~"$service", SourceContext="Microsoft.AspNetCore.Hosting.Diagnostics"}
  | json | ElapsedMilliseconds > 2000
  | unwrap ElapsedMilliseconds [$__range]
) by (RequestPath)
```

**Filtering slow requests for log stream:**
```logql
{service_name=~"$service", SourceContext="Microsoft.AspNetCore.Hosting.Diagnostics"}
| json | ElapsedMilliseconds > 2000
```

### Why two different SourceContexts?

Serilog's ASP.NET Core integration emits logs from two middleware sources with different field names:

| SourceContext | Latency Field | Used For |
|---|---|---|
| `Serilog.AspNetCore.RequestLoggingMiddleware` | `Elapsed` | Status code filtering, error detection |
| `Microsoft.AspNetCore.Hosting.Diagnostics` | `ElapsedMilliseconds` | Latency calculations, slow request tracking |

---

## 📂 Repository Structure

```
├── dashboard/
│   └── erp-loki-dashboard-v2.json   # Grafana dashboard — ready to import
└── README.md
```

---

## 🚀 Import Instructions

1. Open Grafana → **Dashboards** → **New** → **Import**
2. Upload `erp-loki-dashboard-v2.json` or paste its contents
3. Select your **Loki datasource** when prompted
4. Click **Import**

> ⚠️ Requires Grafana 10+ and a running Loki instance with ASP.NET Core logs shipped via Serilog.

---

## ⚙️ Variables

The dashboard exposes two template variables at the top:

| Variable | Type | Purpose |
|---|---|---|
| `$service` | Multi-select query | Filter by `service_name` label — supports `ERPServer.API`, `auth`, or All |
| `$status_code` | Custom dropdown | Filter the log viewer by HTTP status code — All / 200 / 400 / 401 / 404 |

---

## 🗂️ Loki Label Requirements

Your Loki streams must have these labels for the queries to work:

```
service_name   = "ERPServer.API"
SourceContext  = "Serilog.AspNetCore.RequestLoggingMiddleware"
               | "Microsoft.AspNetCore.Hosting.Diagnostics"
```

These are automatically set by Serilog's ASP.NET Core integration and the Loki sink with label extraction configured.

---

## 📸 Dashboard Sections

```
📊 Overview
  └─ Total Requests · Total Errors · 2xx Successes
  └─ 400 Bad Requests · 401 Unauthorized
  └─ 🐌 Slow Requests > 2s (stat + endpoint table + log viewer)

📊 Status Code Distribution
  └─ Donut Pie · Bar Gauge · Filtered Log Viewer

🛣️ Endpoint Intelligence
  └─ Top Endpoints by Count · Slowest Endpoints by Avg Latency

📈 Errors Over Time
  └─ 4xx/5xx/Auth stacked bars · Log Volume by Service

🔍 Log Streams
  └─ Error Logs Full Detail · All Logs Live Stream
```

---

## 💡 Lessons Learned

- **LogQL `unwrap` requires the field to exist** on the specific `SourceContext` — using the wrong source context returns no data silently
- **Instant queries in Grafana** always return `Value #A/B/C` — you must use the **Organize Fields** transformation to rename them properly
- **`avg by (label)` is PromQL syntax** — in LogQL the grouping goes after: `avg_over_time(...) by (label)`
- **`$__range` vs `$__interval`** — use `$__range` for stat/table instant queries, `$__interval` for time series panels
- Building a metrics-equivalent observability layer from pure logs is very achievable with LogQL's metric queries

---

## 🔗 Related

- [Grafana Loki LogQL Docs](https://grafana.com/docs/loki/latest/query/)
- [Serilog.AspNetCore](https://github.com/serilog/serilog-aspnetcore)
- [Grafana Loki Serilog Sink](https://github.com/serilog-contrib/serilog-sinks-grafana-loki)

---

*Built for [IMS Software](https://imssoftware.com.np) ERP infrastructure observability.*
