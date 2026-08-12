# Design Decisions

Why this dashboard looks the way it does. Each section states the decision, the
alternative that was rejected, and the reason.

---

## 1. Layout follows the order of the questions you actually ask

The dashboard reads top to bottom as an escalating investigation:

| Row | Question it answers | Widget type |
|---|---|---|
| Header | Which environment, who do I page? | text |
| KPI tiles | Is it broken *right now*? | singleValue |
| Golden Signals | What kind of broken? | timeSeries |
| Resources | Why? | timeSeries |
| Correlation | What caused what? | timeSeries, dual axis |

**Rejected:** grouping by AWS service (all ALB widgets, then all EC2 widgets).
That mirrors how AWS is organised, not how an incident is diagnosed — it forces
the responder to translate "the site is slow" into "which namespace do I want",
which is exactly the work the dashboard should be doing for them.

**Consequence:** the most decision-relevant information is above the fold. You
should be able to tell whether to escalate without scrolling.

---

## 2. `singleValue` for status, `timeSeries` for everything else

The four tiles at the top are the only `singleValue` widgets on the dashboard.

- **`singleValue`** answers "what is it now?" It is the fastest thing on a screen
  to read, and it is the right choice *only* when the current value alone is
  actionable. It has no history, so it cannot tell you if the number is normal.
- **`timeSeries`** answers "is this normal?" Almost every real monitoring question
  is a comparison against the recent past, so the majority of the dashboard is
  line charts.

**Rejected:** making everything a tile because tiles are compact. A tile showing
"CPU 62%" is unactionable — 62% is fine if it has been 60% all week and alarming
if it was 8% ten minutes ago. Only the chart carries that distinction.

**Rejected:** `gauge` view for CPU/memory. Gauges consume a lot of pixels to
express one number against a fixed range and, like tiles, discard the trend.

---

## 3. Stacked bars for status codes, overlaid lines for latency percentiles

These two widgets sit side by side and use deliberately opposite settings.

- **Errors is `stacked: true`.** The three series are *parts of one whole* —
  every request is exactly one of 2XX, 4XX, 5XX. Stacking makes total height =
  total requests, so the red band's proportion of the bar *is* the error rate,
  readable without arithmetic.
- **Latency percentiles are unstacked.** P50, P95 and P99 are not parts of a
  whole; they are three views of the same distribution. Stacking them would draw
  P95 at P50+P95, which is a number that does not exist and means nothing.

The takeaway: `stacked` is a claim that the series sum to something meaningful.
Using it when they don't produces a chart that is confidently wrong.

---

## 4. Percentiles instead of averages for latency

`p50`, `p95` and `p99` are plotted; `Average` is not on the dashboard at all.

An average latency is dragged down by the fast majority and hides the slow tail
entirely. A service where 95% of requests take 50ms and 5% take 4 seconds
averages out to a perfectly healthy-looking 250ms — while one user in twenty is
staring at a spinner. Percentiles are statements about *users*: P95 = 500ms means
19 out of 20 requests came back within half a second. That maps directly onto an
SLO; an average does not.

---

## 5. SLO thresholds drawn on the chart, not kept in someone's head

The Latency widget carries horizontal annotations at 0.5s (P95) and 1.0s (P99);
CPU carries one at 80%. All use `fill: "above"`, which shades the breach region.

This converts "is 0.47 seconds bad?" — a question requiring institutional
knowledge — into "is the line in the shaded area?", a question answerable by
anyone. It is the single highest-leverage line of JSON on the dashboard, because
it moves the *judgement* into the artefact instead of leaving it with whoever
happened to design the SLO.

---

## 6. Fixed 0–100 axes on percentage metrics

CPU and memory both set `yAxis.left.min: 0, max: 100`.

CloudWatch auto-scales axes to the data by default, which means an idle instance
oscillating between 1.8% and 2.4% renders as a dramatic sawtooth filling the
chart. Pinning the axis to the metric's true range makes visual magnitude
proportional to real severity — a flat line near the floor *looks* like nothing
happening, because nothing is happening.

---

## 7. Dual axis only in the correlation widget

The correlation view is the only chart using per-series `{"yAxis": "right"}`.

Dual axes are genuinely dangerous: two independent scales let you make any two
lines appear to move together, and the apparent correlation is an artefact of the
scaling. It is justified here and only here, because this widget's *entire
purpose* is comparing the shape and timing of curves whose units differ by orders
of magnitude (0.1 seconds vs 100 requests vs 60 percent). Everywhere else, series
on one chart share a unit and share an axis.

---

## 8. Widget size encodes priority

Sizes are not arbitrary:

- Correlation view: **24 x 6** — needs maximum horizontal resolution, since
  correlation is judged by whether two lines bend at the same x position.
- Golden Signals: **12 x 6** — two-up, large enough to read a spike's shape.
- Resource charts: **6 x 5** — four-up, smaller. These are supporting evidence
  consulted *after* a Golden Signal has flagged something.
- KPI tiles: **6 x 3** — short, because a single number needs no vertical space.

Size is a visual-hierarchy channel that costs nothing. Bigger reads as more
important, and here it genuinely is.

---

## 9. Two periods, chosen per metric, not globally

- **60s** — request rate, target health, and the KPI tiles. These change fast and
  the recency matters more than the smoothing.
- **300s** — errors, latency percentiles, and all resource metrics. Wider buckets
  suppress single-datapoint noise so a trend is visible instead of a hairball.

**Note on cost and accuracy:** ALB metrics are emitted at 60s natively; EC2 is
5-minute basic monitoring unless detailed monitoring is enabled (it is enabled on
this instance, which is why CPU has 1-minute granularity available). Requesting a
period finer than the publishing interval yields gaps, not detail.

---

## 10. Metric math for the error-rate tile, raw counts for the errors chart

Lab Part 5 offers replacing the stacked status-code chart with a computed
`(m1/m2)*100` error-rate line. That change was **deliberately not made**, and the
math expression is used on the KPI tile instead.

The reasoning: a ratio is the right thing for the *tile*, because a tile has no
context and a raw count of 50 is uninterpretable without knowing the denominator.
But on the *chart*, the raw counts carry strictly more information — the stacked
view shows the ratio (band proportion) **and** absolute volume **and** the 4XX/5XX
split, all at once. Replacing it with a single percentage line would discard two
of those three for no gain. Both patterns are therefore present on the dashboard,
each where it is the better fit.

---

## 11. Colour used as a signal, never as decoration

`#d13212` red for 5XX and unhealthy hosts, `#ff7f0e` orange for 4XX and warning
thresholds, `#2ca02c` green for 2XX and healthy hosts. Red means "page someone",
orange means "look into it", green means "working as intended", everywhere on the
dashboard without exception.

Consistency is what makes this work. The moment red is used once for a series
that is merely *interesting*, red stops meaning anything anywhere.

---

## Known limitations

- **Single instance.** Every EC2 widget is pinned to one `InstanceId`. For a real
  fleet these become `Search` expressions or aggregate across an Auto Scaling
  group, otherwise the dashboard breaks every time an instance is replaced.
- **Static deployment marker.** The vertical annotation is a hardcoded timestamp.
  In production the CI/CD pipeline would rewrite it via `put-dashboard` on deploy.
- **No alarm-state widget.** The Part 1 mockup shows an "Active Alerts / System
  Status" band. That needs CloudWatch alarms, which belong to a different lab;
  the header text widget stands in for it here.
