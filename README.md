# Lab M6.03 - Build Dashboard for Web Tier Health

**Repository:** [https://github.com/cloud-engineering-bootcamp/ce-lab-build-dashboard-web-tier](https://github.com/cloud-engineering-bootcamp/ce-lab-build-dashboard-web-tier)

**Activity Type:** Individual  
**Estimated Time:** 45 minutes

## Learning Objectives

- [ ] Design effective dashboard layout
- [ ] Create CloudWatch dashboard with multiple widgets
- [ ] Display key metrics (Request Rate, Error Rate, Latency)
- [ ] Use appropriate visualizations
- [ ] Implement correlation views

## Your Task

Build a comprehensive monitoring dashboard for your web tier (Load Balancer + EC2 instances) that provides at-a-glance health status.

**Success Criteria:**
- Dashboard shows Golden Signals (Latency, Traffic, Errors, Saturation)
- Multiple widget types used appropriately
- Clear visual hierarchy
- Dashboard is actionable

## Quick Start

```bash
# 1. Create dashboard
aws cloudwatch put-dashboard \
  --dashboard-name WebTierMonitoring \
  --dashboard-body file://dashboard.json

# 2. View dashboard
# Console → CloudWatch → Dashboards → WebTierMonitoring

# 3. Share dashboard (optional)
# Actions → Share dashboard → Get shareable link
```

## 📤 What to Submit

**Submission Type:** GitHub Repository

Create a **public** GitHub repository named `ce-lab-web-tier-dashboard` containing:

### Required Files

**1. README.md**
- Dashboard design rationale
- Widget explanations
- How to use the dashboard
- Screenshots

**2. Dashboard Configuration** (`config/`)
- `dashboard.json` - Complete dashboard JSON
- `widgets.md` - Explanation of each widget
- `design-decisions.md` - Why you chose specific visualizations

**3. Screenshots** (`screenshots/`)
- Full dashboard view
- Each major section
- Example of using dashboard to identify issue

**4. Documentation** (`docs/`)
- `dashboard-guide.md` - How to read the dashboard
- `troubleshooting.md` - What to look for when investigating issues

### Repository Structure
```
ce-lab-web-tier-dashboard/
├── README.md
├── config/
│   ├── dashboard.json
│   ├── widgets.md
│   └── design-decisions.md
├── screenshots/
│   ├── 01-full-dashboard.png
│   ├── 02-golden-signals.png
│   ├── 03-resource-utilization.png
│   └── 04-correlation-view.png
└── docs/
    ├── dashboard-guide.md
    └── troubleshooting.md
```

## Grading: 100 points

- Dashboard layout and design: 25pts
- Golden Signals implemented: 30pts
- Appropriate visualizations: 20pts
- Documentation: 15pts
- Bonus features: 10pts

## Detailed Instructions

### Part 1: Plan Dashboard Layout (10 min)

**Dashboard Hierarchy:**
```
┌────────────────────────────────────────────┐
│ WEB TIER HEALTH - Production              │ ← Header
├────────────────────────────────────────────┤
│ 🔴 Active Alerts: 2                        │ ← Critical Info
│ System Status: ⚠️ DEGRADED                │
├────────────────────────────────────────────┤
│ [Request Rate]  │  [Error Rate]            │ ← Golden
│                 │                          │   Signals
│ [Latency P95]   │  [Target Health]         │
├────────────────────────────────────────────┤
│ [CPU]  │ [Memory]  │ [Network]  │ [Disk]  │ ← Resources
├────────────────────────────────────────────┤
│ [Request Rate + Error Rate Correlation]    │ ← Correlation
└────────────────────────────────────────────┘
```

**Key Metrics to Include:**
- Request rate (requests/minute)
- Error rate (% and count)
- Latency (P50, P95, P99)
- Target health (healthy target count)
- CPU utilization
- Memory utilization
- Network throughput
- Disk usage

### Part 2: Create Dashboard JSON (15 min)

**Basic Dashboard Structure:**

```json
{
  "widgets": [
    {
      "type": "text",
      "x": 0,
      "y": 0,
      "width": 24,
      "height": 1,
      "properties": {
        "markdown": "# Web Tier Monitoring Dashboard\n**Environment:** Production | **Team:** Cloud Engineering | **On-Call:** [Slack #oncall](https://slack.com/oncall)"
      }
    },
    {
      "type": "metric",
      "x": 0,
      "y": 1,
      "width": 12,
      "height": 6,
      "properties": {
        "title": "Request Rate (per minute)",
        "metrics": [
          ["AWS/ApplicationELB", "RequestCount", {"stat": "Sum", "label": "Total Requests"}]
        ],
        "view": "timeSeries",
        "stacked": false,
        "region": "us-east-1",
        "period": 60,
        "yAxis": {
          "left": {
            "label": "Requests",
            "showUnits": false
          }
        }
      }
    },
    {
      "type": "metric",
      "x": 12,
      "y": 1,
      "width": 12,
      "height": 6,
      "properties": {
        "title": "Error Rate",
        "metrics": [
          ["AWS/ApplicationELB", "HTTPCode_Target_5XX_Count", {"stat": "Sum", "label": "5XX Errors", "color": "#d13212"}],
          [".", "HTTPCode_Target_4XX_Count", {"stat": "Sum", "label": "4XX Errors", "color": "#ff7f0e"}],
          [".", "HTTPCode_Target_2XX_Count", {"stat": "Sum", "label": "2XX Success", "color": "#2ca02c"}]
        ],
        "view": "timeSeries",
        "stacked": false,
        "region": "us-east-1",
        "period": 300
      }
    },
    {
      "type": "metric",
      "x": 0,
      "y": 7,
      "width": 12,
      "height": 6,
      "properties": {
        "title": "Latency (P95)",
        "metrics": [
          ["AWS/ApplicationELB", "TargetResponseTime", {"stat": "p95", "label": "P95 Latency"}]
        ],
        "view": "timeSeries",
        "region": "us-east-1",
        "period": 300,
        "yAxis": {
          "left": {
            "label": "Seconds",
            "showUnits": false
          }
        },
        "annotations": {
          "horizontal": [
            {
              "label": "SLO Target (500ms)",
              "value": 0.5,
              "fill": "above",
              "color": "#d13212"
            }
          ]
        }
      }
    },
    {
      "type": "metric",
      "x": 12,
      "y": 7,
      "width": 12,
      "height": 6,
      "properties": {
        "title": "Healthy Target Count",
        "metrics": [
          ["AWS/ApplicationELB", "HealthyHostCount", {"stat": "Average", "label": "Healthy Targets"}],
          [".", "UnHealthyHostCount", {"stat": "Average", "label": "Unhealthy Targets", "color": "#d13212"}]
        ],
        "view": "timeSeries",
        "region": "us-east-1",
        "period": 60
      }
    }
  ]
}
```

**Complete Dashboard with All Sections:**

```json
{
  "widgets": [
    {
      "type": "text",
      "x": 0,
      "y": 0,
      "width": 24,
      "height": 1,
      "properties": {
        "markdown": "# 🌐 Web Tier Health Dashboard\n**Environment:** Production | **Region:** us-east-1 | **Last Updated:** Auto-refresh every 60s\n\n**On-Call:** [Slack #oncall](https://slack.com) | **Runbook:** [Wiki](https://wiki.company.com)"
      }
    },
    {
      "type": "metric",
      "x": 0,
      "y": 1,
      "width": 6,
      "height": 3,
      "properties": {
        "title": "Current Request Rate",
        "metrics": [
          ["AWS/ApplicationELB", "RequestCount", {"stat": "Sum", "period": 60}]
        ],
        "view": "singleValue",
        "region": "us-east-1"
      }
    },
    {
      "type": "metric",
      "x": 6,
      "y": 1,
      "width": 6,
      "height": 3,
      "properties": {
        "title": "Error Rate (%)",
        "metrics": [
          ["AWS/ApplicationELB", "HTTPCode_Target_5XX_Count", {"id": "m1", "visible": false}],
          [".", "RequestCount", {"id": "m2", "visible": false}],
          [{"expression": "(m1/m2)*100", "label": "Error Rate %", "id": "e1"}]
        ],
        "view": "singleValue",
        "region": "us-east-1",
        "period": 300
      }
    },
    {
      "type": "metric",
      "x": 12,
      "y": 1,
      "width": 6,
      "height": 3,
      "properties": {
        "title": "P95 Latency (ms)",
        "metrics": [
          ["AWS/ApplicationELB", "TargetResponseTime", {"stat": "p95"}]
        ],
        "view": "singleValue",
        "region": "us-east-1",
        "period": 300,
        "setPeriodToTimeRange": true
      }
    },
    {
      "type": "metric",
      "x": 18,
      "y": 1,
      "width": 6,
      "height": 3,
      "properties": {
        "title": "Healthy Targets",
        "metrics": [
          ["AWS/ApplicationELB", "HealthyHostCount", {"stat": "Average"}]
        ],
        "view": "singleValue",
        "region": "us-east-1",
        "period": 60
      }
    },
    {
      "type": "text",
      "x": 0,
      "y": 4,
      "width": 24,
      "height": 1,
      "properties": {
        "markdown": "## 📊 Golden Signals (Last 3 Hours)"
      }
    },
    {
      "type": "metric",
      "x": 0,
      "y": 5,
      "width": 12,
      "height": 6,
      "properties": {
        "title": "Traffic - Request Rate",
        "metrics": [
          ["AWS/ApplicationELB", "RequestCount", {"stat": "Sum", "label": "Requests/min"}]
        ],
        "view": "timeSeries",
        "region": "us-east-1",
        "period": 60,
        "stat": "Sum",
        "yAxis": {
          "left": {
            "label": "Requests"
          }
        }
      }
    },
    {
      "type": "metric",
      "x": 12,
      "y": 5,
      "width": 12,
      "height": 6,
      "properties": {
        "title": "Errors - HTTP Status Codes",
        "metrics": [
          ["AWS/ApplicationELB", "HTTPCode_Target_5XX_Count", {"stat": "Sum", "color": "#d13212"}],
          [".", "HTTPCode_Target_4XX_Count", {"stat": "Sum", "color": "#ff7f0e"}],
          [".", "HTTPCode_Target_2XX_Count", {"stat": "Sum", "color": "#2ca02c"}]
        ],
        "view": "timeSeries",
        "stacked": true,
        "region": "us-east-1",
        "period": 300
      }
    },
    {
      "type": "metric",
      "x": 0,
      "y": 11,
      "width": 12,
      "height": 6,
      "properties": {
        "title": "Latency - Response Time Percentiles",
        "metrics": [
          ["AWS/ApplicationELB", "TargetResponseTime", {"stat": "p50", "label": "P50"}],
          ["...", {"stat": "p95", "label": "P95"}],
          ["...", {"stat": "p99", "label": "P99"}]
        ],
        "view": "timeSeries",
        "region": "us-east-1",
        "period": 300,
        "yAxis": {
          "left": {
            "min": 0,
            "label": "Seconds"
          }
        },
        "annotations": {
          "horizontal": [
            {"value": 0.5, "label": "P95 SLO", "fill": "above", "color": "#ff7f0e"},
            {"value": 1.0, "label": "P99 SLO", "fill": "above", "color": "#d13212"}
          ]
        }
      }
    },
    {
      "type": "metric",
      "x": 12,
      "y": 11,
      "width": 12,
      "height": 6,
      "properties": {
        "title": "Saturation - Target Health",
        "metrics": [
          ["AWS/ApplicationELB", "HealthyHostCount", {"stat": "Average", "color": "#2ca02c"}],
          [".", "UnHealthyHostCount", {"stat": "Average", "color": "#d13212"}]
        ],
        "view": "timeSeries",
        "region": "us-east-1",
        "period": 60
      }
    },
    {
      "type": "text",
      "x": 0,
      "y": 17,
      "width": 24,
      "height": 1,
      "properties": {
        "markdown": "## 💻 EC2 Resource Utilization"
      }
    },
    {
      "type": "metric",
      "x": 0,
      "y": 18,
      "width": 6,
      "height": 5,
      "properties": {
        "title": "CPU Utilization",
        "metrics": [
          ["AWS/EC2", "CPUUtilization", {"stat": "Average"}]
        ],
        "view": "timeSeries",
        "region": "us-east-1",
        "period": 300,
        "yAxis": {
          "left": {
            "min": 0,
            "max": 100,
            "label": "%"
          }
        },
        "annotations": {
          "horizontal": [
            {"value": 80, "label": "Warning", "fill": "above", "color": "#ff7f0e"}
          ]
        }
      }
    },
    {
      "type": "metric",
      "x": 6,
      "y": 18,
      "width": 6,
      "height": 5,
      "properties": {
        "title": "Memory Utilization",
        "metrics": [
          ["CWAgent", "mem_used_percent", {"stat": "Average"}]
        ],
        "view": "timeSeries",
        "region": "us-east-1",
        "period": 300,
        "yAxis": {
          "left": {
            "min": 0,
            "max": 100
          }
        }
      }
    },
    {
      "type": "metric",
      "x": 12,
      "y": 18,
      "width": 6,
      "height": 5,
      "properties": {
        "title": "Network In/Out",
        "metrics": [
          ["AWS/EC2", "NetworkIn", {"stat": "Sum", "label": "Network In"}],
          [".", "NetworkOut", {"stat": "Sum", "label": "Network Out"}]
        ],
        "view": "timeSeries",
        "region": "us-east-1",
        "period": 300
      }
    },
    {
      "type": "metric",
      "x": 18,
      "y": 18,
      "width": 6,
      "height": 5,
      "properties": {
        "title": "Disk Usage",
        "metrics": [
          ["CWAgent", "disk_used_percent", {"stat": "Average"}]
        ],
        "view": "timeSeries",
        "region": "us-east-1",
        "period": 300
      }
    },
    {
      "type": "text",
      "x": 0,
      "y": 23,
      "width": 24,
      "height": 1,
      "properties": {
        "markdown": "## 🔍 Correlation View (for Root Cause Analysis)"
      }
    },
    {
      "type": "metric",
      "x": 0,
      "y": 24,
      "width": 24,
      "height": 6,
      "properties": {
        "title": "Latency vs Request Rate vs Error Rate vs CPU",
        "metrics": [
          ["AWS/ApplicationELB", "TargetResponseTime", {"stat": "p95", "yAxis": "left", "label": "Latency P95 (ms)"}],
          [".", "RequestCount", {"stat": "Sum", "yAxis": "right", "label": "Request Rate"}],
          [".", "HTTPCode_Target_5XX_Count", {"stat": "Sum", "yAxis": "right", "label": "5XX Errors", "color": "#d13212"}],
          ["AWS/EC2", "CPUUtilization", {"stat": "Average", "yAxis": "right", "label": "CPU %"}]
        ],
        "view": "timeSeries",
        "region": "us-east-1",
        "period": 300
      }
    }
  ]
}
```

### Part 3: Create Dashboard (5 min)

```bash
# Save dashboard JSON to file
cat > dashboard.json <<'EOF'
{
  "widgets": [
    ... (paste full JSON from above)
  ]
}
EOF

# Create dashboard
aws cloudwatch put-dashboard \
  --dashboard-name WebTierMonitoring \
  --dashboard-body file://dashboard.json

# Verify
aws cloudwatch list-dashboards
```

### Part 4: Test and Refine (10 min)

**View Dashboard:**
1. Go to CloudWatch → Dashboards
2. Click "WebTierMonitoring"
3. Verify all widgets load
4. Check time range (default last 3 hours)

**Test Correlation:**
1. Generate load on your application
2. Observe how metrics correlate
3. Note patterns (e.g., high load → high CPU → high latency)

**Refine Layout:**
- Adjust widget sizes
- Reorder for better flow
- Add/remove metrics as needed
- Test on mobile view

### Part 5: Add Advanced Features (Optional) (5 min)

**Add Annotations:**
```json
"annotations": {
  "horizontal": [
    {
      "label": "SLO Threshold",
      "value": 500,
      "fill": "above",
      "color": "#ff0000"
    }
  ],
  "vertical": [
    {
      "label": "Deployment",
      "value": "2024-01-20T14:00:00Z",
      "color": "#00ff00"
    }
  ]
}
```

**Add Math Expressions:**
```json
"metrics": [
  ["AWS/ApplicationELB", "HTTPCode_Target_5XX_Count", {"id": "m1", "visible": false}],
  [".", "RequestCount", {"id": "m2", "visible": false}],
  [{"expression": "(m1/m2)*100", "label": "Error Rate %", "id": "e1"}]
]
```

## Reflection Questions

Answer these in your README:

1. **Why show P95 instead of average latency?**
   - Hint: User experience vs system average

2. **What correlation patterns indicate problems?**
   - Hint: High request rate + high CPU + high latency

3. **Why group related metrics?**
   - Hint: Faster root cause analysis

4. **How many metrics is too many on one dashboard?**
   - Hint: Balance completeness with comprehension

5. **When would you create multiple dashboards?**
   - Hint: Different audiences, different purposes

## Bonus Challenges

**+5 points each:**
- [ ] Add custom business metrics (orders/min, revenue)
- [ ] Create mobile-friendly version
- [ ] Add alarm state widgets
- [ ] Implement drill-down dashboards
- [ ] Add deployment markers

## Troubleshooting

**Issue: Widgets showing "No data"**
- Verify metric namespace and name
- Check dimensions (LoadBalancer ID, Instance ID)
- Ensure metrics are being published
- Check time range

**Issue: Dashboard looks cluttered**
- Reduce to 6-10 key metrics
- Group related metrics
- Use larger widgets
- Consider multiple dashboards

## Resources

- [CloudWatch Dashboards](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Dashboards.html)
- [Dashboard Body Structure](https://docs.aws.amazon.com/AmazonCloudWatch/latest/APIReference/CloudWatch-Dashboard-Body-Structure.html)
- [Math Expressions](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/using-metric-math.html)

---

**Excellent work on your monitoring dashboard!** 📊
