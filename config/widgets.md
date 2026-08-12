# Widget Reference

Every widget in `config/dashboard.json`, in the order it appears, with what it
measures and why it is on the dashboard.

Real dimension values baked into the JSON:

| Placeholder | Real value |
|---|---|
| `YOUR_LB_DIMENSION` | `app/logging-lab-alb/f3f577325c1f0df6` |
| `YOUR_TARGETGROUP_DIMENSION` | `targetgroup/logging-lab-tg/8fc5121b8174ca13` |
| `YOUR_INSTANCE_ID` | `i-06208cc8863bd3bbc` |
| Region | `us-east-1` |

---

## Row 0 — Header (y=0)

### Web Tier Health Dashboard
- **Type:** `text` (markdown), 24 x 1
- **Contents:** environment, region, refresh cadence, and links to the on-call
  Slack channel and the runbook.
- **Why:** a responder who opens this dashboard at 3am needs to know *which*
  environment they are looking at and *where to escalate* before they read a
  single number. Costs one grid row and removes an entire class of mistake.

---

## Row 1 — KPI tiles (y=1)

Four `singleValue` tiles, 6 columns each. These answer "is it broken right now?"
in under two seconds. No trend, no history — just the current number.

### Current Request Rate
- **Metric:** `AWS/ApplicationELB RequestCount`, `Sum`, period 60
- **Reads as:** requests in the most recent one-minute bucket.
- **Why a tile:** traffic volume is the first thing you check. A zero here means
  nothing is reaching the load balancer, which changes the entire investigation.

### Error Rate (%)
- **Metrics:** `HTTPCode_Target_5XX_Count` (`id: m1`, hidden) and
  `RequestCount` (`id: m2`, hidden), plus the math expression
  `(m1/m2)*100` (`id: e1`, "Error Rate %")
- **Reads as:** percentage of requests the target answered with a 5XX.
- **Why a ratio, not a count:** 50 errors is a catastrophe at 100 req/min and
  background noise at 100,000 req/min. Only the ratio is comparable across
  traffic levels, so only the ratio belongs on a KPI tile.
- **`visible: false`** hides the two raw inputs so the tile shows one number
  instead of three.

### P95 Latency (ms)
- **Metric:** `TargetResponseTime`, `p95`, period 300, `setPeriodToTimeRange: true`
- **Reads as:** the 95th percentile response time across the *whole* selected
  time range rather than just the latest 5-minute bucket, which makes the tile
  stable instead of flickering on every refresh.

### Healthy Targets
- **Metric:** `HealthyHostCount`, `Average`, period 60
- **Dimensions:** needs **both** `LoadBalancer` and `TargetGroup` — host-count
  metrics are emitted per target group, and without the second dimension the
  widget renders `--`.
- **Reads as:** how many targets are currently in service. Capacity in one number.

---

## Row 2 — Golden Signals (y=4 divider, y=5 and y=11 charts)

A markdown divider, then four `timeSeries` widgets at 12 x 6 — one per Golden
Signal. Two-up so each chart is wide enough to read a spike's shape.

### Traffic — Request Rate
- **Metric:** `RequestCount`, `Sum`, period 60
- **Why:** the denominator for everything else. A latency change means something
  completely different when traffic tripled than when traffic was flat.

### Errors — HTTP Status Codes
- **Metrics:** 5XX (`#d13212` red), 4XX (`#ff7f0e` orange), 2XX (`#2ca02c` green),
  all `Sum`, period 300, **`stacked: true`**
- **Why stacked:** stacking makes the total bar height equal total requests, so
  the red band's *share* of the bar is the error rate, read visually. Unstacked,
  a 5XX line of 3 next to a 2XX line of 3000 is invisible against the axis.
- **Why the traffic-light colours:** severity is legible before the legend is read.

### Latency — Response Time Percentiles
- **Metrics:** `TargetResponseTime` at `p50`, `p95`, `p99`, period 300
- **Annotations:** horizontal SLO lines at `0.5s` (P95, orange) and `1.0s`
  (P99, red), both with `fill: "above"` so breaching the SLO shades the chart.
- **Why three percentiles:** the *gap between them* is the diagnosis. P50 flat
  with P99 climbing means a subset of requests is suffering — a slow dependency,
  a hot shard, one bad host. All three rising together means systemic saturation.

### Saturation — Target Health
- **Metrics:** `HealthyHostCount` (green) and `UnHealthyHostCount` (red),
  `Average`, period 60, both dimensions
- **Why:** capacity over time. Plotting both on one chart makes a failover
  obvious — the green line steps down exactly as the red line steps up.

---

## Row 3 — EC2 Resource Utilization (y=17 divider, y=18 charts)

Four `timeSeries` widgets at 6 x 5. Deliberately smaller than the Golden Signals
above: these are *causes*, consulted after a signal has already told you
something is wrong.

### CPU Utilization
- **Metric:** `AWS/EC2 CPUUtilization`, `Average`, period 300
- **Axis:** pinned `min: 0, max: 100` so the shape of the line reflects real
  severity instead of auto-scaling a 2% wobble to look like a crisis.
- **Annotations:**
  - horizontal `80` "Warning", `fill: "above"`
  - vertical `2026-08-12T07:45:00Z` "Deployment" (green) — the instance launch,
    used as the deploy marker so you can see whether a change lines up with a
    CPU or latency shift.

### Memory Utilization
- **Metric:** `CWAgent mem_used_percent`, `Average`, period 300, axis 0–100
- **Note:** requires the CloudWatch agent from Lab M6.02. EC2 does not publish
  memory natively — the hypervisor cannot see inside the guest OS.

### Network In/Out
- **Metrics:** `NetworkIn` and `NetworkOut`, `Sum`, period 300
- **Why together:** the ratio between them is the signal. In-heavy is upload or
  scrape traffic, out-heavy is normal serving; a sudden inversion is worth a look.

### Disk Usage
- **Metric:** `CWAgent disk_used_percent`, `Average`, period 300
- **Why:** the slowest-moving metric here and the one that takes a service down
  hardest. A full disk fails writes and logging simultaneously.

---

## Row 4 — Correlation View (y=23 divider, y=24 chart)

### Latency vs Request Rate vs Error Rate vs CPU
- **Type:** `timeSeries`, full-width 24 x 6, period 300
- **Metrics:** P95 latency on the **left** axis; request rate, 5XX count and CPU
  on the **right** axis, via the per-series `{"yAxis": "right"}` option.
- **Why full width:** correlation is judged by whether two lines bend at the same
  x position. The wider the chart, the finer the time resolution the eye can
  resolve.
- **Why dual axis:** latency is ~0.1 seconds and request rate is ~100. On one
  axis, latency is a flat line on the floor. Splitting the axes puts both curves
  in the visible band so their *shapes* can be compared, which is the only thing
  this widget is for.
- **How to use it:** find the moment latency bent, then look left-to-right at the
  same timestamp on the other three series. See `docs/troubleshooting.md`.
