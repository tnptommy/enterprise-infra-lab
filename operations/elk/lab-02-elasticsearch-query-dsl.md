# Lab 2 — Elasticsearch Search & Query DSL

A hands-on exercise for querying Elasticsearch directly via its REST API, using the real indices already populated by the lab (`suricata-*`, `winlogbeat-*`, `metricbeat-*`). Kibana's Discover and Lens both translate UI clicks into these same queries under the hood — understanding the raw Query DSL makes it possible to build filters Kibana's point-and-click UI can't express directly, and to debug why a Kibana panel is (or isn't) returning the expected data.

---

## Prerequisites

- Elasticsearch running on LOG01 with data already ingested (confirm via `docs/17`'s verification steps)
- `curl` and `jq` available (install `jq` if missing: `sudo dnf install -y jq`)

---

## Step 1 — The simplest possible query: match_all

```bash
curl -k -u elastic:'<password>' "https://192.168.10.50:9200/suricata-*/_search?pretty" -H 'Content-Type: application/json' -d '
{
  "query": { "match_all": {} },
  "size": 3
}'
```

This returns the first 3 documents Elasticsearch finds, in no particular order — `match_all` is a placeholder, not a real filter. It's the query equivalent of leaving Kibana's search bar empty.

## Step 2 — Filter by an exact field value (term query)

```bash
curl -k -u elastic:'<password>' "https://192.168.10.50:9200/suricata-*/_search?pretty" -H 'Content-Type: application/json' -d '
{
  "query": {
    "term": { "event_type": "alert" }
  },
  "size": 5
}'
```

`term` matches the field's exact stored value — this is what Kibana generates for `event_type: "alert"` typed into the search bar as KQL. Using `term` on a text field that was analyzed (tokenized) rather than kept as a `keyword` will silently return nothing, which is one of the most common Elasticsearch beginner mistakes — confirm a field's exact type first:

```bash
curl -k -u elastic:'<password>' "https://192.168.10.50:9200/suricata-*/_mapping?pretty" | jq '.[] | .mappings.properties.event_type'
```

## Step 3 — Range query (numeric/date filtering)

```bash
curl -k -u elastic:'<password>' "https://192.168.10.50:9200/suricata-*/_search?pretty" -H 'Content-Type: application/json' -d '
{
  "query": {
    "bool": {
      "must": [
        { "term": { "event_type": "alert" } }
      ],
      "filter": [
        { "range": { "@timestamp": { "gte": "now-24h" } } }
      ]
    }
  },
  "size": 0,
  "aggs": {
    "by_severity": {
      "terms": { "field": "alert.severity" }
    }
  }
}'
```

This is the direct Query DSL equivalent of the "Alerts by Severity" Kibana Lens panel built earlier — a `bool` query combining a `must` clause (relevance-scored) with a `filter` clause (not scored, faster, used for anything that's a yes/no condition like a time range), plus an `aggs` block that buckets results by `alert.severity` the same way the Lens pie chart does internally.

> **`must` vs `filter` — why it matters beyond just style:** clauses inside `filter` are cached and skip relevance scoring entirely, which is both faster and the semantically correct choice for anything that isn't "how well does this match" (a time range either applies or it doesn't — there's no partial match to score). Reach for `filter` by default; only use `must` when relevance ranking genuinely matters.

## Step 4 — Full-text search vs exact match

```bash
curl -k -u elastic:'<password>' "https://192.168.10.50:9200/winlogbeat-*/_search?pretty" -H 'Content-Type: application/json' -d '
{
  "query": {
    "match": { "message": "logon failed" }
  },
  "size": 5
}'
```

`match` (unlike `term`) analyzes the search text and the field the same way, splitting into individual words and matching any document containing them — this is what powers Kibana's free-text search bar. Compare this result set to a `match_phrase` query (same syntax, replace `match` with `match_phrase`) to see the difference: `match` finds documents containing "logon" and "failed" anywhere, `match_phrase` requires them adjacent and in that order.

## Step 5 — Aggregation without any query (pure analytics)

```bash
curl -k -u elastic:'<password>' "https://192.168.10.50:9200/metricbeat-*/_search?pretty" -H 'Content-Type: application/json' -d '
{
  "size": 0,
  "aggs": {
    "avg_cpu_by_host": {
      "terms": { "field": "host.name" },
      "aggs": {
        "avg_cpu": { "avg": { "field": "system.cpu.total.pct" } }
      }
    }
  }
}'
```

`"size": 0` tells Elasticsearch not to bother returning any raw documents — only the aggregation result. This is exactly the query shape behind the "System Resource Trend" Lens panel (a nested aggregation: bucket by host, then average within each bucket).

## Step 6 — Counting without fetching anything

```bash
curl -k -u elastic:'<password>' "https://192.168.10.50:9200/suricata-*/_count?pretty" -H 'Content-Type: application/json' -d '
{
  "query": { "term": { "event_type": "alert" } }
}'
```

This is the same query shape used by the `problem.get`-style HTTP agent item built for the Zabbix dashboard — `_count` is cheaper than `_search` when only a number is needed, not the documents themselves.

---

## Exercises to try on your own

1. Write a query against `suricata-*` that returns alerts where `src_ip` starts with `192.168.10.` — use a `wildcard` or `prefix` query on the `.keyword` sub-field, and compare performance/behavior against a `term` query on the same field.
2. Combine a `range` (last 7 days) with a `terms` aggregation on `agent.name`, sorted by count descending, limited to the top 5 — this is the exact query behind "Top 10 Hosts by Alerts," just capped lower.
3. Query `_mapping` on `iis-*` and confirm whether `url_path` exists as a field yet (it won't, until the Grok fix from Lab 1 is applied to the production pipeline) — this is a faster way to check than opening Kibana Discover each time.

## Relevance to the dashboards already built

Every Lens/Visualize panel built for the Zabbix, Wazuh, and Kibana dashboards compiles down to a query shaped like the ones above. When a panel shows unexpected results, the fastest debugging path is often to reconstruct the equivalent raw query here and inspect the actual response — Kibana's UI doesn't always make it obvious which clause is filtering out data.
