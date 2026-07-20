# Lab 1 — Logstash Pipeline Fundamentals

A hands-on exercise for practicing Logstash's `input → filter → output` model on a disposable pipeline, separate from the production pipeline already running on LOG01 (see [`docs/15`](../../docs/15-log01-elasticsearch-logstash-kibana.md)). The goal is to build muscle memory for grok patterns and conditional filtering before touching the real pipeline again — mistakes here don't risk breaking live ingestion from WEB01/WINAPP01/DC01.

---

## Prerequisites

- Logstash already built and running on LOG01 (from `docs/15`)
- SSH access to LOG01
- A working Elasticsearch instance with valid credentials (the same `elastic` user used throughout the lab)

---

## Step 1 — Create a scratch log file to parse

Rather than risking the real Suricata/Winlogbeat/IIS logs, generate a small synthetic log in a format close to a standard Linux auth log:

```bash
mkdir -p ~/elk-lab
cat > ~/elk-lab/sample-auth.log << 'EOF'
Jul 20 09:12:03 web01 sshd[1044]: Failed password for root from 203.0.113.44 port 51422 ssh2
Jul 20 09:12:07 web01 sshd[1044]: Failed password for root from 203.0.113.44 port 51430 ssh2
Jul 20 09:13:15 web01 sshd[1051]: Accepted password for deploy from 192.168.10.100 port 44210 ssh2
Jul 20 09:14:02 web01 sudo: deploy : TTY=pts/0 ; PWD=/home/deploy ; USER=root ; COMMAND=/usr/bin/systemctl restart nginx
EOF
```

## Step 2 — Write a standalone pipeline config

Keep this in a separate directory from the real pipeline (`/etc/logstash/conf.d/`), so it never accidentally gets picked up by the running service:

```bash
mkdir -p ~/elk-lab/pipeline
cat > ~/elk-lab/pipeline/lab1.conf << 'EOF'
input {
  file {
    path => "/root/elk-lab/sample-auth.log"
    start_position => "beginning"
    sincedb_path => "/dev/null"
  }
}

filter {
  grok {
    match => { "message" => "%{SYSLOGTIMESTAMP:log_timestamp} %{SYSLOGHOST:host} %{DATA:process}(?:\[%{POSINT:pid}\])?: %{GREEDYDATA:log_message}" }
  }

  if [log_message] =~ "Failed password" {
    mutate { add_field => { "event_outcome" => "failure" } }
    grok {
      match => { "log_message" => "Failed password for %{USERNAME:target_user} from %{IP:source_ip} port %{NUMBER:source_port}" }
    }
  } else if [log_message] =~ "Accepted password" {
    mutate { add_field => { "event_outcome" => "success" } }
    grok {
      match => { "log_message" => "Accepted password for %{USERNAME:target_user} from %{IP:source_ip} port %{NUMBER:source_port}" }
    }
  }

  date {
    match => [ "log_timestamp", "MMM dd HH:mm:ss" ]
    target => "@timestamp"
  }
}

output {
  elasticsearch {
    hosts => ["https://192.168.10.50:9200"]
    user => "elastic"
    password => "<elastic-superuser-password>"
    ssl_certificate_authorities => ["/opt/logstash/config/certs/http_ca.crt"]
    index => "lab1-auth-%{+YYYY.MM.dd}"
  }
  stdout { codec => rubydebug }
}
EOF
```

**What each block is doing, and why it's structured this way:**
- The first `grok` splits every line into its generic syslog shape (`timestamp`, `host`, `process`, `pid`, `message`) — this always matches, regardless of what the process actually logged.
- The conditional (`if [log_message] =~ ...`) is a second, more specific `grok` pass that only runs when the message body matches a known pattern — this is the standard way to layer parsing: generic first, specific second, rather than writing one giant regex that tries to handle every case at once.
- `date` overwrites Logstash's default `@timestamp` (which would otherwise be "when Logstash read the line") with the actual time the event happened according to the log itself — this matters for any timeline visualization built later.

## Step 3 — Run it in test mode first

Always validate config syntax before running for real — this catches typos without touching Elasticsearch:

```bash
sudo /opt/logstash/bin/logstash -f ~/elk-lab/pipeline/lab1.conf --config.test_and_exit
```

## Step 4 — Run the pipeline

```bash
sudo /opt/logstash/bin/logstash -f ~/elk-lab/pipeline/lab1.conf
```

Watch the `stdout { codec => rubydebug }` output in the terminal — confirm `event_outcome`, `target_user`, `source_ip` appear as separate fields, not buried inside the raw `message` string. Stop with `Ctrl+C` once you've seen all 4 lines processed.

## Step 5 — Verify in Elasticsearch directly

```bash
curl -k -u elastic:'<password>' "https://192.168.10.50:9200/lab1-auth-*/_search?pretty&size=10"
```

Confirm 4 documents exist, and that `source_ip` for the two failed attempts is `203.0.113.44` — not a value Logstash invented, but one correctly extracted from the raw text.

## Step 6 — Clean up

This is scratch data — don't leave it cluttering the cluster:

```bash
curl -k -u elastic:'<password>' -X DELETE "https://192.168.10.50:9200/lab1-auth-*"
rm -rf ~/elk-lab/pipeline/lab1.conf
```

---

## Exercises to try on your own

1. Add a third conditional branch that catches `sudo:` lines and extracts `USER=`, `COMMAND=` into their own fields.
2. Add a `geoip` filter on `source_ip` (Logstash ships this plugin by default) and confirm a `geoip.country_name` field appears for `203.0.113.44`.
3. Deliberately break the grok pattern (e.g. typo `%{IP:source_ip}` as `%{IIP:source_ip}`) and observe what Logstash does with unmatched lines — check for a `_grokparsefailure` tag in the output. Recognizing this tag is the single most useful skill for debugging a real pipeline that "isn't working."

## Relevance to the production pipeline on LOG01

The real pipeline (`docs/15`) doesn't need a `grok` filter for Suricata (it's already structured JSON via the `ndjson` parser in Filebeat), but IIS logs currently land in Elasticsearch unparsed — see the note in [`operations/elk/elk-reference-material.md`](./elk-reference-material.md). The pattern practiced in Step 2 above (generic grok first, conditional specific grok second) is exactly the shape needed to finally parse those IIS logs into `url_path`, `status_code`, etc.
