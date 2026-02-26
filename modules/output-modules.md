# BBot Output Modules

Complete reference for all output, reporting, and data export modules. Output modules receive all events and write them to various destinations.

Output modules are specified with `-om` (or `--output-modules`).

---

## stdout

**Description:** The default output module. Prints events to the terminal in a human-readable format with color coding. Always active when running BBOT interactively.

**Events:** All events
**Format:** Human-readable colored text

```bash
# Always active by default
bbot -t example.com -f subdomain-enum
```

---

## json

**Description:** Writes all events to a NDJSON (newline-delimited JSON) file. Each line is a complete JSON object representing one event. The primary format for programmatic processing.

**Output File:** `output.ndjson` (in scan directory)

**Options:**
| Option | Default | Description |
|---|---|---|
| `output_file` | `{scan_dir}/output.ndjson` | Custom output file path |
| `append` | `False` | Append to existing file instead of overwriting |
| `siem_friendly` | `False` | SIEM-compatible format with flattened fields |

```bash
# JSON is auto-enabled; also specify for custom path
bbot -t example.com -f subdomain-enum -om json \
     -o /opt/scans/example/

# SIEM-friendly format
bbot -t example.com -f subdomain-enum -om json \
     -c output_modules.json.siem_friendly=true
```

**JSON Event Structure:**
```json
{
  "type": "DNS_NAME",
  "id": "uuid-here",
  "data": "api.example.com",
  "scope_distance": 0,
  "scan": "scan_name",
  "timestamp": "2026-02-25T12:00:00",
  "resolved_hosts": ["192.168.1.1"],
  "tags": ["resolved", "a-record"],
  "module": "crt",
  "module_sequence": "crt"
}
```

---

## csv

**Description:** Writes events to a CSV (comma-separated values) file. Suitable for import into spreadsheets, Excel, or database tools. One row per event with flattened fields.

**Output File:** `output.csv`

**Options:**
| Option | Default | Description |
|---|---|---|
| `output_file` | `{scan_dir}/output.csv` | Custom output file path |

```bash
bbot -t example.com -f subdomain-enum -om csv \
     -o /opt/scans/example/
```

**CSV Columns:**
`type`, `data`, `scope_distance`, `scan`, `timestamp`, `tags`, `module`

---

## txt

**Description:** Writes a simple text file — one data value per line. The cleanest format for piping into other tools (subfinder → amass → bbot → nmap pattern).

**Output Files:**
- `subdomains.txt` — DNS_NAME events
- `urls.txt` — URL events
- `emails.txt` — EMAIL_ADDRESS events

```bash
bbot -t example.com -f subdomain-enum -om txt \
     -o /opt/scans/example/

# Use output directly
cat /opt/scans/example/scan_name/subdomains.txt | httprobe
```

---

## subdomains

**Description:** Dedicated output module for subdomain text files. Writes only DNS_NAME events to a clean list — one subdomain per line.

**Output File:** `subdomains.txt`

```bash
bbot -t example.com -f subdomain-enum -om subdomains

# Extract subdomains from latest scan
ls -t ~/bbot/scans/*/subdomains.txt | head -1 | xargs cat
```

---

## emails

**Description:** Dedicated output module for email addresses. Writes all EMAIL_ADDRESS events to a clean text file — one email per line.

**Output File:** `emails.txt`

```bash
bbot -t example.com -p email-enum -om emails

# Use in targeted attacks
cat emails.txt | while read email; do echo "$email"; done
```

---

## web_parameters

**Description:** Writes discovered web parameters (from paramminer_* modules) to a file. One parameter per line with context.

**Output File:** `web_parameters.txt`

```bash
bbot -t example.com -f web-paramminer -om web_parameters
```

---

## sqlite

**Description:** Writes all events to a SQLite database file. Enables SQL queries over scan results without requiring a database server. Excellent for offline analysis.

**Output File:** `output.sqlite`

**Options:**
| Option | Default | Description |
|---|---|---|
| `output_file` | `{scan_dir}/output.sqlite` | Custom SQLite file path |

```bash
bbot -t example.com -f subdomain-enum -om sqlite \
     -o /opt/scans/example/

# Query results
sqlite3 output.sqlite "SELECT data FROM events WHERE type='DNS_NAME'"
sqlite3 output.sqlite "SELECT data FROM events WHERE type='VULNERABILITY'"
sqlite3 output.sqlite "SELECT data, module FROM events WHERE type='FINDING'"
```

**Useful SQLite Queries:**
```sql
-- All subdomains
SELECT data FROM events WHERE type='DNS_NAME';

-- All URLs
SELECT data FROM events WHERE type='URL';

-- All vulnerabilities
SELECT data, module FROM events WHERE type='VULNERABILITY';

-- Event type counts
SELECT type, COUNT(*) FROM events GROUP BY type ORDER BY COUNT(*) DESC;

-- Findings by module
SELECT module, COUNT(*) FROM events WHERE type='FINDING' GROUP BY module;

-- Open ports
SELECT data FROM events WHERE type='OPEN_TCP_PORT';
```

---

## postgres

**Description:** Streams all events directly to a PostgreSQL database in real-time. Ideal for integration with dashboards, analysis platforms, and multi-scan result aggregation.

**Options:**
| Option | Default | Description |
|---|---|---|
| `host` | `"localhost"` | PostgreSQL host |
| `port` | `5432` | PostgreSQL port |
| `database` | `"bbot"` | Database name |
| `username` | `"bbot"` | Database username |
| `password` | `""` | Database password |
| `table` | `"events"` | Table name to write events to |

```bash
bbot -t example.com -f subdomain-enum -om postgres \
     -c output_modules.postgres.host=db.example.com \
        output_modules.postgres.database=bbot_results \
        output_modules.postgres.username=bbot \
        output_modules.postgres.password=secret
```

---

## mysql

**Description:** Streams events to a MySQL/MariaDB database.

**Options:**
| Option | Default | Description |
|---|---|---|
| `host` | `"localhost"` | MySQL host |
| `port` | `3306` | MySQL port |
| `database` | `"bbot"` | Database name |
| `username` | `"bbot"` | MySQL username |
| `password` | `""` | MySQL password |
| `table` | `"events"` | Table to write to |

```bash
bbot -t example.com -f subdomain-enum -om mysql \
     -c output_modules.mysql.host=localhost \
        output_modules.mysql.password=secret
```

---

## neo4j

**Description:** Streams events to a Neo4j graph database. Enables relationship-based analysis of discovered assets — visualize how domains, IPs, URLs, and findings connect to each other as a knowledge graph.

**Options:**
| Option | Default | Description |
|---|---|---|
| `url` | `"bolt://localhost:7687"` | Neo4j Bolt connection URL |
| `username` | `"neo4j"` | Neo4j username |
| `password` | `""` | Neo4j password |

```bash
bbot -t example.com -f subdomain-enum -om neo4j \
     -c output_modules.neo4j.url=bolt://localhost:7687 \
        output_modules.neo4j.password=password

# Query in Neo4j
# MATCH (n:DNS_NAME) RETURN n.data LIMIT 25
# MATCH (n)-[:PARENT]->(p) RETURN n.data, p.data
```

---

## splunk

**Description:** Forwards events to a Splunk HTTP Event Collector (HEC) endpoint in real-time. For integration with SOC/SIEM workflows.

**Options:**
| Option | Default | Description |
|---|---|---|
| `url` | `""` | Splunk HEC URL (e.g., `https://splunk:8088/services/collector`) |
| `token` | `""` | Splunk HEC token |
| `index` | `"bbot"` | Splunk index to write to |
| `source` | `"bbot"` | Splunk source value |
| `sourcetype` | `"bbot:event"` | Splunk sourcetype |

```bash
bbot -t example.com -f subdomain-enum -om splunk \
     -c output_modules.splunk.url=https://splunk.corp:8088/services/collector \
        output_modules.splunk.token=HEC_TOKEN \
        output_modules.splunk.index=security_recon
```

---

## http

**Description:** Sends events to an HTTP endpoint (webhook) via POST requests. Enables integration with any custom application or SIEM that accepts webhooks.

**Options:**
| Option | Default | Description |
|---|---|---|
| `url` | `""` | HTTP POST endpoint URL |
| `authorization` | `""` | Authorization header value |
| `headers` | `{}` | Additional custom headers |
| `method` | `"POST"` | HTTP method |

```bash
bbot -t example.com -f subdomain-enum -om http \
     -c output_modules.http.url=https://api.yourapp.com/bbot/events \
        output_modules.http.authorization="Bearer TOKEN"
```

---

## websocket

**Description:** Streams events via WebSocket to a connected client. Enables real-time dashboards and live scan monitoring.

**Options:**
| Option | Default | Description |
|---|---|---|
| `url` | `""` | WebSocket server URL (`ws://` or `wss://`) |
| `token` | `""` | Authentication token |

```bash
bbot -t example.com -f subdomain-enum -om websocket \
     -c output_modules.websocket.url=ws://localhost:8080/events \
        output_modules.websocket.token=SECRET
```

---

## slack

**Description:** Sends event notifications to a Slack channel via webhook. Useful for alerting on high-severity findings during long-running scans.

**Options:**
| Option | Default | Description |
|---|---|---|
| `webhook_url` | `""` | Slack Incoming Webhook URL |
| `event_types` | `["VULNERABILITY","FINDING"]` | Event types to send to Slack |
| `min_severity` | `"HIGH"` | Minimum severity for vulnerability alerts |

```bash
bbot -t example.com -f web-thorough -om slack \
     -c output_modules.slack.webhook_url=https://hooks.slack.com/services/T.../B.../... \
        output_modules.slack.min_severity=CRITICAL
```

---

## discord

**Description:** Sends event notifications to a Discord channel via webhook.

**Options:**
| Option | Default | Description |
|---|---|---|
| `webhook_url` | `""` | Discord Webhook URL |
| `event_types` | `["VULNERABILITY","FINDING"]` | Event types to send |
| `min_severity` | `"HIGH"` | Minimum severity for alerts |

```bash
bbot -t example.com -f web-thorough -om discord \
     -c output_modules.discord.webhook_url=https://discord.com/api/webhooks/.../... \
        output_modules.discord.min_severity=HIGH
```

---

## teams

**Description:** Sends event notifications to Microsoft Teams via an Incoming Webhook connector.

**Options:**
| Option | Default | Description |
|---|---|---|
| `webhook_url` | `""` | Teams Incoming Webhook URL |
| `event_types` | `["VULNERABILITY","FINDING"]` | Event types to send |

```bash
bbot -t example.com -f web-thorough -om teams \
     -c output_modules.teams.webhook_url=https://outlook.office.com/webhook/.../...
```

---

## web_report

**Description:** Generates a self-contained HTML web report of all scan findings. Includes charts, statistics, asset tables, and a filterable findings section. Best human-readable output format.

**Output File:** `web_report.html`

**Options:**
| Option | Default | Description |
|---|---|---|
| `output_file` | `{scan_dir}/web_report.html` | Report output path |

```bash
bbot -t example.com -f web-thorough -om web_report \
     -o /opt/reports/example/

# Open report
xdg-open /opt/reports/example/scan_name/web_report.html
```

---

## asset_inventory

**Description:** Generates a comprehensive asset inventory report. Summarizes all discovered assets (domains, IPs, URLs, technologies, open ports) in a structured format suitable for asset management.

**Output File:** `asset-inventory.csv`

**Options:**
| Option | Default | Description |
|---|---|---|
| `output_file` | `{scan_dir}/asset-inventory.csv` | Inventory file path |
| `use_previous` | `True` | Merge with previous scan's inventory |

```bash
bbot -t example.com -f subdomain-enum web-basic -om asset_inventory \
     -o /opt/inventories/example/
```

---

## nmap_xml

**Description:** Writes port scan results in Nmap XML format. Enables import into tools that consume Nmap output (Metasploit, Nessus, etc.).

**Output File:** `output.nmap.xml`

```bash
bbot -t example.com -m portscan -om nmap_xml \
     -o /opt/scans/example/

# Import into Metasploit
# db_import /opt/scans/example/scan_name/output.nmap.xml
```

---

## python

**Description:** Python module output — advanced integration. Enables custom Python code to receive events via the module system.

**For Developers Only:** Requires extending the BasePythonOutput class.

---

## Output Module Combinations

### Bug Bounty Standard Output
```bash
bbot -t example.com \
     -f subdomain-enum web-basic \
     -om json csv subdomains web_report \
     -o ~/bug_bounty/company/bbot_scans/
```

### SIEM Integration
```bash
bbot -t example.com \
     -f subdomain-enum web-thorough \
     -om json splunk \
     -c output_modules.splunk.url=https://splunk:8088/services/collector \
        output_modules.splunk.token=TOKEN \
        output_modules.json.siem_friendly=true
```

### Real-time Alerting + Persistence
```bash
bbot -t example.com \
     -f web-thorough \
     -om sqlite slack subdomains emails \
     -c output_modules.slack.webhook_url=WEBHOOK \
        output_modules.slack.min_severity=HIGH \
     -o /opt/scans/
```

### Full Export Suite
```bash
bbot -t example.com \
     -f kitchen-sink -ef deadly \
     -om json csv sqlite subdomains emails \
        web_report asset_inventory nmap_xml \
     -o /opt/full_scan/
```

### Database + Graph Analysis
```bash
bbot -t example.com \
     -f subdomain-enum web-thorough \
     -om postgres neo4j json \
     -c output_modules.postgres.host=db.local \
        output_modules.postgres.password=secret \
        output_modules.neo4j.url=bolt://graph.local:7687 \
        output_modules.neo4j.password=secret
```
