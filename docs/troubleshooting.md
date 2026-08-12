# Troubleshooting Guide

What to look for when the dashboard says something is wrong, and what to do about
the dashboard itself when *it* is what's broken.

---

## Part 1 — Investigating an incident

### The general method

1. **Start at the tiles.** Which Golden Signal is out of range? That decides
   everything downstream.
2. **Check Traffic first, always.** Every other signal is interpreted relative to
   load. Latency doubling under 3x traffic is a capacity story; latency doubling
   under flat traffic is a defect story. Same chart, opposite conclusions.
3. **Establish when it started.** Drag-select on any chart to zoom; all widgets
   follow. Find the first bend, not the worst point — the worst point is usually
   downstream damage.
4. **Go to the Correlation View** and read down that timestamp. Whichever series
   moved *first* is the lead.
5. **Only then look at resources.** CPU/memory/disk explain *why*; they are
   evidence, not detection.

### Symptom → likely cause

#### High latency, traffic flat, CPU flat
The instance is not working harder, so it is **waiting** on something.

- Look at: P50 vs P99 spread on the Latency chart
- Narrow spread (everything slow) → a shared dependency: database, cache, an
  external API
- Wide spread (P99 only) → one code path, one slow query, one bad host
- **Not** a scale-up problem. Adding capacity to a system that is idle-waiting
  changes nothing

#### High latency, traffic up, CPU up
Straightforward saturation — you are serving more than this instance can handle.

- Confirm: all three percentiles rising together, CPU near or above the 80% line
- Expect errors to follow as timeouts start firing
- Fix: scale out, or shed load. This is the one case where more capacity is the
  actual answer

#### 5XX errors, traffic and CPU normal
The application is failing requests while perfectly healthy at the infrastructure
level. Almost always a code or configuration problem.

- Check the **Deployment** marker on the CPU widget — did errors begin at that
  vertical line? If so, roll back first and diagnose after
- Check Target Health: if hosts are flapping, requests are being dropped mid-flight
- Correlate with application logs (Lab M6.01/M6.02 log group) at the same timestamp

#### 4XX climbing, 5XX flat
Your service is behaving correctly and rejecting bad requests.

- A caller deployed a breaking change, a credential expired, or someone is
  scanning for URLs that do not exist
- Rarely your outage — but confirm it is not a mis-routed health check, which
  shows up as regular, evenly-spaced 4XXs

#### Healthy target count dropped
- Green steps down, red steps up → targets failing `/health`
- Green steps down, red stays 0 → targets deregistered or terminated (scaling
  event, instance replacement)
- **Watch what happens next:** remaining targets now absorb the same traffic, so
  latency and CPU will rise on the survivors. That secondary spike is a
  consequence, not a second incident

#### Traffic dropped to zero
The problem is **upstream of the load balancer**, not in the application.

- Check DNS, the listener, security groups, and the client side
- A perfectly healthy application with zero traffic looks identical to a healthy
  application at 3am. Confirm this is unexpected before escalating

#### Memory climbing steadily, never falling
A leak. Time-to-impact is the slope — extend the range to 1w to project it.

- A sudden cliff back to baseline is a restart, meaning something already crashed
  and came back. Check whether error rate spiked at the same moment

#### Disk approaching 100%
Unhandled, this takes the service down hard: writes fail and logging fails at the
same time, so you lose the telemetry right when you need it.

- Usually log rotation that stopped working, or a runaway temp directory
- Extend to 1w to see the slope and estimate the deadline

---

## Part 2 — When the dashboard itself is broken

### A widget shows "No data available" or `--`

Work through in this order:

1. **Region.** Every widget hardcodes `"region": "us-east-1"`. If the resources
   live elsewhere, every widget loads empty while the JSON is otherwise perfect.
   This is the most common cause by a wide margin.
2. **Dimensions.** Confirm the values in the JSON still exist:
   ```bash
   aws elbv2 describe-load-balancers --query 'LoadBalancers[].LoadBalancerArn'
   aws ec2 describe-instances \
     --filters "Name=tag:Name,Values=logging-lab" \
     --query 'Reservations[].Instances[].InstanceId'
   ```
   A recreated instance gets a **new** `InstanceId`, and every EC2/CWAgent widget
   silently goes blank. Same for a recreated ALB.
3. **Both LB dimensions on host-count metrics.** `HealthyHostCount` and
   `UnHealthyHostCount` need `LoadBalancer` **and** `TargetGroup`. With only the
   first they render `--` and look like an outage.
4. **Is the metric being published at all?**
   ```bash
   aws cloudwatch list-metrics --namespace CWAgent \
     --dimensions Name=InstanceId,Value=<instance-id>
   ```
   Empty result → the CloudWatch agent is not running or not configured for those
   metrics (Lab M6.02 Part 3).
5. **Time range.** Metrics only exist for periods that had activity. `RequestCount`
   produces no datapoints when nothing hit the load balancer — that is an absence
   of traffic, not an absence of monitoring.
6. **Period finer than the publishing interval.** Requesting 60s on a metric
   published every 300s yields gaps, not resolution.

### All ALB widgets empty, EC2 widgets fine
Expected when there is no load balancer — the lab treats this as acceptable. If
there *is* one, check that traffic actually went **through** it: curling
`localhost:5000` on the instance never touches the ALB and produces zero ALB
metrics.

```bash
ALB_DNS=$(aws elbv2 describe-load-balancers \
  --names logging-lab-alb --query 'LoadBalancers[0].DNSName' --output text)
curl http://$ALB_DNS/
```

### Target never becomes healthy
Before touching security groups, confirm the application is actually running.
This is by far the most common cause:

```bash
curl http://localhost:5000/health     # on the instance
systemctl status webapp
```

If the app was started in the foreground of an SSH session, it died when that
session closed. It needs to run under systemd or `nohup ... & disown`.

Then, in order: instance SG allows :5000 from the ALB SG, ALB SG allows :80
inbound, health check path is `/health`, and the check needs *several consecutive*
successes — give it a minute or two.

### `put-dashboard` fails to parse the file
On Windows the AWS CLI decodes `file://` with the system codepage and chokes on
the emoji in the header widget:

```
Error parsing parameter '--dashboard-body': ... could not be decoded
```

```powershell
$env:AWS_CLI_FILE_ENCODING = "utf-8"
```

### Dashboard edits do not appear
A dashboard is only ever the JSON last sent by `put-dashboard`. Editing
`config/dashboard.json` locally has no effect until it is pushed again. There is
no sync step.

### Metric math tile shows blank or `NaN`
`(m1/m2)*100` divides by zero when `RequestCount` is 0 for the period. A blank
error-rate tile during a traffic gap is arithmetic, not an outage — cross-check
against the Traffic chart before believing it.
