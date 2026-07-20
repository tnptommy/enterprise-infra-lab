# Lab 5 — Securing and Optimizing the ELK Stack

Securing and optimizing an ELK cluster is usually covered at a conceptual level (secure the cluster, tune resources, plan for scale) rather than as a single fixed hands-on task — this lab turns those themes into concrete exercises against LOG01, including one directly informed by an incident already hit in this environment.

---

## Prerequisites

- Elasticsearch/Logstash/Kibana running on LOG01 with security enabled (`docs/15`)
- `elastic` superuser credentials

---

## Part A — Access control: stop using the superuser everywhere

Every query and Beat config across this lab so far authenticates as `elastic` — the single superuser account. That's fine for initial setup, but it means any leaked credential (a `.yml` file committed by accident, a shoulder-surfed terminal) has full cluster admin rights. Real deployments create narrowly-scoped roles instead.

### Step 1 — Create a read-only role for dashboard/query use

In Kibana: **Stack Management → Roles → Create role**, name it `lab_readonly`, and grant:
```
Indices: suricata-*, winlogbeat-*, iis-*, metricbeat-*, packetbeat-*
Privileges: read, view_index_metadata
```

### Step 2 — Create a user with that role

```bash
curl -k -u elastic:'<password>' -X POST "https://192.168.10.50:9200/_security/user/lab_viewer" -H 'Content-Type: application/json' -d '
{
  "password": "<a-new-password>",
  "roles": ["lab_readonly"],
  "full_name": "Lab read-only viewer"
}'
```

### Step 3 — Confirm the restriction actually works

```bash
# Should succeed — read access to an allowed index
curl -k -u lab_viewer:'<password>' "https://192.168.10.50:9200/suricata-*/_count"

# Should fail with a 403 — no write privilege
curl -k -u lab_viewer:'<password>' -X DELETE "https://192.168.10.50:9200/suricata-*"

# Should fail — no access to the .security index or cluster admin actions
curl -k -u lab_viewer:'<password>' "https://192.168.10.50:9200/_security/user"
```

A role that doesn't visibly block *something* hasn't actually been tested — Step 3's failing cases matter as much as the succeeding one.

---

## Part B — Index Lifecycle Management: don't let data grow forever

Every index in this lab (`suricata-*`, `winlogbeat-*`, etc.) has been growing without any retention policy since `docs/17` — left alone, this eventually repeats the disk-pressure problem already seen on LOG01/LOG02 during earlier stages of this lab, just for Elasticsearch instead of the JVM heap.

### Step 4 — Check current index sizes

```bash
curl -k -u elastic:'<password>' "https://192.168.10.50:9200/_cat/indices?v&s=store.size:desc"
```

### Step 5 — Create an ILM policy: hot for 7 days, then delete after 30

```bash
curl -k -u elastic:'<password>' -X PUT "https://192.168.10.50:9200/_ilm/policy/lab-retention-policy" -H 'Content-Type: application/json' -d '
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": { "rollover": { "max_age": "7d", "max_primary_shard_size": "5gb" } }
      },
      "delete": {
        "min_age": "30d",
        "actions": { "delete": {} }
      }
    }
  }
}'
```

### Step 6 — Attach it to one index pattern as a test

Rather than applying this to every production index immediately, test it against a disposable index first (reuse the `lab1-auth-*` pattern from Lab 1 if it still exists, or create a fresh scratch index) — confirm the policy attaches and shows a phase in **Stack Management → Index Lifecycle Policies** before rolling it out to `suricata-*`/`winlogbeat-*`/etc. for real.

---

## Part C — Resource sizing: connecting this back to a real incident

This lab's own history is the most concrete "optimization" lesson available: `wazuh-indexer`'s JVM was killed by the Linux OOM killer during a `make -j$(nproc)` build elsewhere on MON01, because JVM heap was sized without headroom for anything else running on the box (see the Wazuh Manager troubleshooting notes referenced from `docs/13`).

### Step 7 — Check current JVM heap allocation on LOG01's Elasticsearch

```bash
cat /etc/elasticsearch/jvm.options.d/heap.options 2>/dev/null || grep -E "^-Xm[sx]" /etc/elasticsearch/jvm.options
free -h
```

### Step 8 — Apply the standard sizing rule and explain why

Elastic's own guidance is heap = 50% of available RAM, capped at ~31GB (beyond that, JVM pointer compression stops working and the extra heap is wasted) — check whether LOG01's current setting follows this:
```bash
# Example: on an 8GB VM, heap should be ~4GB, not more
-Xms4g
-Xmx4g
```
Leaving the *other* half of RAM to the OS matters because Elasticsearch relies heavily on the OS-level filesystem cache for read performance, not just JVM heap — a heap sized too large actually **hurts** performance by starving that cache, not just risking an OOM kill.

### Step 9 — If a change is needed, apply and verify

```bash
sudo systemctl restart elasticsearch
curl -k -u elastic:'<password>' "https://192.168.10.50:9200/_nodes/stats/jvm?pretty" | grep -A3 heap_used_percent
```

---

## Exercises to try on your own

1. Repeat Part C's heap-sizing check against `wazuh-indexer` on MON01 — apply the same 50%-of-RAM rule there, and compare against whatever value led to the OOM kill in the first place.
2. Create a second Kibana role, `lab_fim_analyst`, scoped only to `wazuh-alerts-*` with a filter restricting it to documents where `rule.groups: "syscheck"` — this is how a real SOC would give a junior analyst FIM-only visibility without exposing every other alert type.
3. Set up **Snapshot Lifecycle Management** (a companion feature to ILM) pointing at a local filesystem repository, and confirm a scheduled snapshot actually completes — this is the backup half of the retention story ILM alone doesn't cover.

## Relevance to the rest of this lab

Part A and B are genuinely new ground — nothing in `docs/12`–`docs/17` sets up role-based access or retention policies. Part C isn't new territory conceptually (the same "don't assume default sizing is safe" lesson already learned the hard way with Wazuh Indexer), but formalizing it as a checked exercise here makes it something to verify on **every** JVM-backed service in this lab (Elasticsearch, Logstash, Wazuh Indexer, Wazuh Manager where applicable) rather than something fixed once and forgotten.
