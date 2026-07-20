# Lab 4 — System Monitoring with Metricbeat, and a Load Test

This lab pairs Metricbeat-collected system metrics with a synthetic load test (`stress --cpu`) to watch the effect show up live in Kibana, run against WEB01's already-installed Metricbeat (`docs/17` Step 10) instead of installing a fresh instance.

---

## Prerequisites

- Metricbeat already running on WEB01, shipping to LOG01 (`docs/17` Step 10)
- A `Metricbeat` Data View already created in Kibana (`docs/17` Step 14)
- `stress` installed on WEB01 (`sudo dnf install -y stress` — EPEL, already enabled per the [golden baseline](../../docs/05-golden-baseline-rocky-linux-10.md#step-5--apply-package-baseline))

---

## Step 1 — Confirm the baseline, before creating any load

```bash
curl -k -u elastic:'<password>' "https://192.168.10.50:9200/metricbeat-*/_search?pretty" -H 'Content-Type: application/json' -d '
{
  "size": 1,
  "sort": [ { "@timestamp": "desc" } ],
  "query": { "term": { "host.name": "WEB01" } }
}' | grep -A1 '"system.cpu.total.pct"'
```

Note the current value — this is WEB01 essentially idle, and is the number every later reading gets compared against.

## Step 2 — Build a combined dashboard (metric + log side by side)

This is the series' specific instruction: one dashboard, two panels from two different data sources, viewed together rather than switching tabs.

1. New Dashboard → **Create visualization**
2. Panel 1 — Line chart, Data view `Metricbeat`: `@timestamp` on X, `Average of system.cpu.total.pct` on Y, filtered to `host.name: "WEB01"`
3. Panel 2 — Data table, Data view `Suricata`: `alert.signature.keyword` in Rows, Count in Metric, filtered to `event_type: "alert"` — a live view of what WEB01 is currently seeing, alongside how hard its CPU is working
4. Save the dashboard, set the refresh interval (top-right) to `10 seconds` so both panels update without manual refreshing during the next step

## Step 3 — Generate load and watch it live

On **WEB01**:
```bash
stress --cpu 8 --timeout 60s
```

Keep the dashboard open in a browser during this — the CPU line should climb visibly within the next couple of 10-second refresh cycles, then fall back to baseline once the 60-second timeout ends.

## Step 4 — Confirm numerically, not just visually

```bash
curl -k -u elastic:'<password>' "https://192.168.10.50:9200/metricbeat-*/_search?pretty" -H 'Content-Type: application/json' -d '
{
  "size": 0,
  "query": {
    "bool": {
      "filter": [
        { "term": { "host.name": "WEB01" } },
        { "range": { "@timestamp": { "gte": "now-2m" } } }
      ]
    }
  },
  "aggs": {
    "max_cpu": { "max": { "field": "system.cpu.total.pct" } }
  }
}'
```
Confirm `max_cpu` from the last 2 minutes is well above the Step 1 baseline — this is the same "trust the raw query over the chart" habit practiced in Lab 2 and Lab 3.

---

## Exercises to try on your own

1. Add a third panel to the dashboard: `system.load.1` (1-minute load average) alongside CPU % — during the stress test, compare how quickly each metric rises and falls; load average lags CPU % slightly, which is a useful thing to know when reading real incident timelines later.
2. Repeat the stress test with `stress --vm 4 --vm-bytes 512M --timeout 60s` (memory pressure instead of CPU) and confirm `system.memory.actual.used.pct` reacts while `system.cpu.total.pct` stays comparatively flat — this is a concrete demonstration of why the Zabbix dashboard (see `operations/zabbix/`) tracks CPU and Memory as separate widgets rather than one combined "load" number.
3. Set up a Kibana **Alert** (Stack Management → Rules) that fires when `system.cpu.total.pct` exceeds `0.8` for WEB01, sustained for 1 minute — then re-run Step 3 and confirm it triggers. This is the natural next step beyond "watching a dashboard," toward the kind of automated detection Wazuh already does for security events (`operations/wazuh/`).

## Relevance to the rest of this lab

This same load-test-and-observe pattern is the standard way to sanity-check *any* monitoring pipeline before trusting it during a real incident — it's worth repeating against the Zabbix and Wazuh dashboards too (e.g. trigger a CPU threshold on MON01 deliberately, confirm the Gauge widget and the Problems panel both react) rather than assuming a dashboard that looks reasonable at idle will also behave correctly under load.
