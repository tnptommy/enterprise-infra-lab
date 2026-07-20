# Lab 3 — Kibana Lens Visualization Practice

A structured practice sequence for Kibana's Lens editor, using it deliberately on a few different chart types before returning to extend the real "Log Analytics Overview" dashboard. Lens's drag-and-drop model rewards muscle memory more than reading about it — this lab is meant to be typed through, not just skimmed.

---

## Prerequisites

- Kibana running on LOG01, reachable and authenticated
- At least one Data View already created (`Suricata`, `Winlogbeat`, or `Metricbeat` — see `docs/17` Step 14)

---

## Step 1 — A single metric, the simplest Lens panel

`Visualize → Create visualization → Lens`

1. Data view: `Suricata`
2. Chart type: `Metric`
3. Drag `Records` into the **Metric** slot
4. Set the filter bar to `event_type: "alert"`

Confirm the number matches what Step 6 of Lab 2's `_count` query returned for the same time range — this is a useful habit: whenever a Kibana panel's number looks surprising, cross-check it against the raw Elasticsearch query rather than assuming the panel is wrong or right.

## Step 2 — A breakdown (Pie), and why field type matters

1. Change chart type to `Pie`
2. Drag `alert.severity` into **Slice by**
3. Click the newly-added `alert.severity` bucket to expand its settings — set **Number of values** to `10`, **Order by** to `Count of records, Descending`

Try the same exercise with a text field instead of a numeric one — drag `agent.name.keyword` into Slice by on a fresh Pie chart. Notice Lens requires the `.keyword` sub-field, not the plain `agent.name` — this is the same underlying reason the `term` query in Lab 2 Step 2 needed an unanalyzed field: Lens's bucket aggregations need an exact value to group on, not a tokenized text field.

## Step 3 — A time series (Bar), and interval behavior

1. New Lens, Data view: `Metricbeat`
2. Chart type: `Bar vertical`
3. Drag `@timestamp` into **Horizontal axis**
4. Drag `system.cpu.total.pct` into **Vertical axis**, then click it and change aggregation to `Average`
5. Set the time range to `Last 7 days`, then to `Last 1 hour` — watch the horizontal axis interval change automatically between the two (this is Lens's "Auto" interval behavior, choosing a bucket size that keeps the number of bars readable regardless of range)

## Step 4 — Breaking down a time series by a dimension

1. Continue from Step 3
2. Drag `host.name` into **Breakdown**
3. Click it, set **Number of values** to `5`

This produces multiple colored series on the same chart — the same shape as the "Alerts Timeline by Host" panel already built, just against different source data. Switch the chart's Stacked/Unstacked toggle (top of the right panel) and observe the difference: stacked shows total volume with per-host contribution, unstacked overlays each host's line independently, better for comparing shapes rather than totals.

## Step 5 — Lens's Suggestions panel

1. Start a fresh Lens visualization
2. Drag just `@timestamp` and `Records` in — nothing else
3. Look at the **Suggestions** row at the bottom of the screen — Lens proposes several chart types (Bar, Line, Area) based only on the two fields dropped in

Click through 2-3 suggestions without reconfiguring anything manually — this is often faster than manually picking a chart type first, especially early in exploring an unfamiliar field.

## Step 6 — A Data table with a calculated ranking

1. Data view: `Suricata`, filter `event_type: "alert"`
2. Chart type: `Data table`
3. Drag `src_ip.keyword` into **Rows**
4. Drag `Records` into **Metric**
5. Click the `src_ip.keyword` row config — set **Number of values** to `10`, **Order by** to `Count of records, Descending`

Add the KQL filter `and src_ip: "192.168.10.*"` and observe how much the table changes — this reproduces the exact fix applied to the "Top Source IPs" panel in the real dashboard (filtering out link-local/IPv6 noise).

---

## Exercises to try on your own

1. Build a Line chart with **two** Y-axis metrics at once (e.g. `Average system.cpu.total.pct` and `Average system.memory.actual.used.pct` on the same chart) — confirm whether Lens puts them on one shared axis or offers a second axis, and when you'd want each.
2. Build a Data table with a calculated column using Lens's **Formula** feature (e.g. a percentage of total) rather than a raw aggregation — this is the Lens equivalent of Elasticsearch's `bucket_script` aggregation.
3. Save one of the visualizations built in this lab to the library, then open the same saved object in **Discover** via "Explore in Discover" (top-right link) — compare what the same underlying query looks like as a document list versus a chart.

---

## Capstone exercise — build a 2-chart dashboard and confirm it updates live

This capstone exercise combines a category breakdown with a time series into a single dashboard, then proves the pipeline is genuinely live rather than a static snapshot — a standard verification pattern worth repeating on any new dashboard, not just this lab exercise.

1. Create a new **Dashboard** (not just a Lens visualization) containing at least two panels:
   - A **Bar** chart: `event_type` (or `alert.severity`) on the horizontal axis, Count of records on the vertical — "events by category," the same shape as the series' "log by level" bar chart.
   - A **Line** chart: `@timestamp` on the horizontal axis (Auto interval), Count of records on the vertical — "events over time," the same shape as the series' "log by timestamp" line chart.
2. Save the dashboard.
3. On WEB01, generate a new event the two charts should reflect — the simplest option already available in this lab is triggering Suricata's test rule:
   ```bash
   ping -c 4 192.168.10.21
   ```
4. Return to the dashboard and refresh (or wait for the auto-refresh interval if one is set) — confirm the Bar chart's relevant category count increases by 1, and the Line chart's most recent bucket ticks up.
5. If nothing changes, work backward through the pipeline rather than assuming the dashboard is broken: check `_cat/indices` (Lab 2, Step 5's approach) to confirm the document actually landed in Elasticsearch first, before suspecting Kibana.

This confirms the full loop end-to-end — event generated → shipped → indexed → queried → rendered — in a single observable action, which is a more convincing test than any individual step in isolation.

## Relevance to the dashboards already built

Every technique here maps directly onto a panel already in the "Log Analytics Overview" dashboard (see `operations/elk/` design notes) — Step 2 is "Suricata Alerts by Severity," Step 4 is "Events Timeline (All Sources)," Step 6 is "Top Source IPs." Practicing them isolated, one at a time, makes it much faster to extend or fix that dashboard later without re-deriving each pattern from scratch under pressure while something is actually broken.
