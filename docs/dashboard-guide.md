# Dashboard Guide — How to Read It

Open: **CloudWatch → Dashboards → WebTierMonitoring** (us-east-1)

---

## The 10-second read

Look only at the four tiles at the top. In order:

| Tile | Healthy | Investigate | Page someone |
|---|---|---|---|
| **Current Request Rate** | within normal range for the hour | ±50% vs the same time yesterday | `0` with no known maintenance |
| **Error Rate (%)** | < 0.1% | 0.1% – 1% | > 1% |
| **P95 Latency** | < 0.5s | 0.5s – 1s | > 1s sustained |
| **Healthy Targets** | = expected target count | one below expected | 0, or fewer than half |

If all four are in the healthy column, the web tier is fine. Stop reading.

> A blank tile (`--`) is **not** "healthy" — it means no data. See
> `troubleshooting.md`.

---

## The 1-minute read

Scroll to **Golden Signals** and look at the *shape* of each chart, not the
absolute numbers.

### Traffic — Request Rate
Establishes the denominator for everything below. Check this first: a change in
any other signal means something entirely different depending on whether traffic
moved with it.

- Flat and steady → any latency or error change is a **system** problem
- Sharp climb → the system may simply be under more load than it can serve
- Sudden drop to zero → traffic is not arriving. The problem is upstream (DNS,
  the load balancer, the client), *not* in the application

### Errors — HTTP Status Codes
Stacked, so total bar height is total requests and each band's share is its rate.

- **Green (2XX) dominant** → normal
- **Orange (4XX) band growing** → clients sending bad requests: a broken deploy
  of a caller, an expired credential, someone scanning for URLs that don't exist.
  Usually not your outage, but worth knowing
- **Red (5XX) band visible at all** → your service is failing requests. This is
  the band that pages people
- **Total height collapses** → not an error problem, a traffic problem; go back
  to the Traffic chart

### Latency — Response Time Percentiles
Three lines, and the **gap between them** is the diagnosis:

- **All three low and tight** → healthy
- **P50 flat, P99 climbing** → a *subset* of requests is slow. Think one bad
  host, a hot partition, a slow dependency on one code path. Most users are fine
- **All three rising together** → systemic. The whole service is saturated —
  check the resource row
- **Any line inside the shaded region** → the SLO is being breached right now.
  The shading is the threshold; no memory required

### Saturation — Target Health
- **Green flat at expected count, red at zero** → healthy
- **Green steps down** → you lost capacity. Remaining targets now absorb the same
  traffic, so expect latency to rise shortly after
- **Red climbing** → targets are failing their health check. The app is up but
  not answering `/health`, or has crashed
- **Green at 0** → complete outage. Nothing is serving traffic

---

## The deep dive

### EC2 Resource Utilization
Consulted *after* a Golden Signal has told you something is wrong. These explain
**why**, they do not tell you **whether**.

- **CPU** — sustained above the shaded 80% line means compute-bound; latency will
  follow. The green vertical "Deployment" line marks a code change: if CPU steps
  up *at* that line, the deploy is your suspect
- **Memory** — a slow steady climb that never falls back is a leak. A cliff back
  to baseline is a process restart, i.e. something died
- **Network In/Out** — the ratio is the signal. Out-heavy is normal serving;
  a sudden inversion to in-heavy suggests uploads, scraping, or an attack
- **Disk** — moves slowly and fails hard. Anything on a steady climb toward 100%
  is a scheduled outage you have not scheduled yet

### Correlation View
The full-width chart at the bottom. Left axis is P95 latency; right axis carries
request rate, 5XX count, and CPU. **The lines are on different scales — compare
their shape and timing, never their height.**

Read it in one move: find the moment latency bent upward, then look straight down
the same timestamp at the other three series. Whichever moved first is your lead.

---

## Time ranges

Default is 3 hours. Change it in the top-right.

| Range | Use for |
|---|---|
| **1h** | active incident — maximum resolution |
| **3h** | default; enough context to see whether "now" is unusual |
| **1d** | did this start with the morning deploy? |
| **1w** | capacity trends, slow leaks, disk growth |

Zoom by dragging horizontally across any chart — **all widgets follow**, which is
the whole point of putting these metrics on one dashboard.

---

## Recreating or updating the dashboard

```bash
aws cloudwatch put-dashboard \
  --dashboard-name WebTierMonitoring \
  --dashboard-body file://config/dashboard.json
```

`put-dashboard` overwrites in place — it is the only apply step. Editing
`config/dashboard.json` locally changes nothing until this command is re-run.

On Windows/PowerShell the CLI decodes `file://` using the system codepage and
will reject the emoji in the header widget. Set the encoding first:

```powershell
$env:AWS_CLI_FILE_ENCODING = "utf-8"
```

To point the dashboard at different infrastructure, substitute the three
dimension values (see the table in `config/widgets.md`):

```bash
sed -i "s|app/logging-lab-alb/f3f577325c1f0df6|$LB_DIMENSION|g; \
        s|targetgroup/logging-lab-tg/8fc5121b8174ca13|$TG_DIMENSION|g; \
        s|i-06208cc8863bd3bbc|$INSTANCE_ID|g" config/dashboard.json
```
