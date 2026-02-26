---
name: bbot
description: |-
  BBOT bug bounty reconnaissance skill. Executes intelligent BBOT scans that
  stay within program scope, respect rate limits, and configure modules based
  on bug bounty program policies. Covers 117+ modules across DNS, port scanning,
  web fuzzing, vulnerability scanning, cloud storage, and code repository discovery.
license: MIT
compatibility: MONK TAO skills system, Claude Code
metadata:
  author: shart123456
  usage: skill file for MONK TAO skills system
  version: 0.1.0
  related files: ""
  creation date: 2026-02-26
  last modified: 2026-02-26
---
# BBOT Skill

## How and When to Trigger This Skill

**HARD REQUIREMENTS:**
- Load this skill BEFORE executing any BBOT scan or reconnaissance task.
- Do NOT attempt BBOT scans from general knowledge — module flags, config options, and scope rules require this skill's context.

### Quick Activation Patterns
- `"Run BBOT scan for [company]"`
- `"BBOT scan [target]"`
- `"Start reconnaissance on [target]"`
- `"Enumerate subdomains for [target]"`
- `"Check for subdomain takeover on [target]"`
- `"Scan for S3 buckets / cloud assets for [company]"`
- `"Bug Bounty and BBot"` + program page
- `"Run nuclei / dnsbrute / httpx / ffuf"` against a target

### Keywords That Trigger This Skill
bbot, recon, reconnaissance, subdomain enumeration, dnsbrute, httpx, nuclei, ffuf, bug bounty, attack surface, s3 bucket, cloud enum, subdomain takeover, baddns, passive recon, portscan, crt.sh, sslcert, trufflehog

### Anti-Patterns (Do NOT activate for)
- `"scan this PDF"` (document processing, not BBOT)
- `"scan for viruses"` (malware scanning, not network recon)
- `"enumerate array items"` (programming, not security)
- `"subdomain of a URL"` without a scan request (general knowledge)

---

## Overview

**PURPOSE:** Provide Claude with full context to execute BBOT scans intelligently — selecting the right modules, flags, rate limits, and scope controls for any bug bounty engagement.

**PROCEDURE**
1. Understand the user request and target
2. Extract scope — identify in-scope domains, out-of-scope blacklist
3. Select scan type based on program policy (passive / safe-active / thorough / full)
4. Load the appropriate workflow
5. Build the BBOT command with correct flags, modules, and output options
6. Execute and capture output
7. Parse results and surface key findings

**BASIC PRINCIPLES**
1. Always define scope before scanning — never run without a blacklist if out-of-scope assets exist
2. Match aggressiveness to program policy — passive for VDPs, active for permissive programs
3. Rate limit by default — set `web_requests_per_second` unless explicitly told otherwise
4. Output to organized directories — `~/bug_bounty/$COMPANY/bbot_scans/`

---

## Architecture Overview

### Core Components
- **SKILL.md** — Skill definition, triggers, and workflow routing (this file)
- **workflows/** — Individual scan workflows for each engagement type
- **documentation/** — Full BBOT module reference, technical config reference, and module category deep-dives
- **configs/** — API key templates and custom preset YAML files
- **data/** — Wordlists and target scope files
- **templates/** — Output report templates
- **cauldron/** — Runtime scan artifacts and results

---

## Included Workflows

### 1. passive_recon.md
**Purpose:** Zero-contact reconnaissance using certificate transparency, passive DNS, and API sources only
**Triggers:** `"passive recon"`, `"no active scanning"`, `"passive only"`, `"VDP scope"`
**Instructions:** Complete step-by-step instructions for executing this workflow are here: `workflows/passive_recon.md`

### 2. safe_active_recon.md
**Purpose:** Light active scanning — subdomain brute force, HTTP probing, port scan, safe vuln checks
**Triggers:** `"safe active"`, `"standard recon"`, `"active scanning allowed"`, `"enumerate subdomains"`
**Instructions:** Complete step-by-step instructions for executing this workflow are here: `workflows/safe_active_recon.md`

### 3. full_engagement.md
**Purpose:** Comprehensive 8-phase engagement — passive intel through visual survey, all safe-active modules
**Triggers:** `"full engagement"`, `"comprehensive scan"`, `"everything"`, `"full recon"`
**Instructions:** Complete step-by-step instructions for executing this workflow are here: `workflows/full_engagement.md`

### 4. subdomain_takeover.md
**Purpose:** Detect dangling CNAME chains and DNS zone takeover candidates using baddns suite
**Triggers:** `"subdomain takeover"`, `"check takeovers"`, `"baddns"`, `"dangling CNAME"`
**Instructions:** Complete step-by-step instructions for executing this workflow are here: `workflows/subdomain_takeover.md`

### 5. vuln_scan.md
**Purpose:** Nuclei, badsecrets, retirejs, baddns against live hosts — focused vulnerability detection
**Triggers:** `"vuln scan"`, `"run nuclei"`, `"check vulnerabilities"`, `"CVE scan"`
**Instructions:** Complete step-by-step instructions for executing this workflow are here: `workflows/vuln_scan.md`

### 6. cloud_hunt.md
**Purpose:** Enumerate S3, GCS, Azure Blob, Firebase storage buckets and Azure tenant/realm discovery
**Triggers:** `"cloud hunt"`, `"S3 buckets"`, `"cloud assets"`, `"storage buckets"`, `"Azure enum"`
**Instructions:** Complete step-by-step instructions for executing this workflow are here: `workflows/cloud_hunt.md`

### 7. web_app_audit.md
**Purpose:** Parameter mining, secret detection, web fuzzing, SSRF checks, WAF fingerprinting
**Triggers:** `"web app audit"`, `"parameter mining"`, `"web fuzzing"`, `"ffuf"`, `"web thorough"`
**Instructions:** Complete step-by-step instructions for executing this workflow are here: `workflows/web_app_audit.md`

### 8. code_leak_hunt.md
**Purpose:** GitHub, GitLab, Postman, Trufflehog — discover leaked source code and hardcoded secrets
**Triggers:** `"code leak"`, `"github secrets"`, `"trufflehog"`, `"postman secrets"`, `"leaked credentials"`
**Instructions:** Complete step-by-step instructions for executing this workflow are here: `workflows/code_leak_hunt.md`

---

## Pre-Scan Scope Extraction

Before running any scan, define scope variables:

```bash
COMPANY="acme"
TARGET="acme.com"
BLACKLIST="admin.acme.com staging.acme.com legacy.acme.com"

# Dry run to confirm scope
bbot -t $TARGET --blacklist $BLACKLIST -f subdomain-enum passive --dry-run
```

**Scope decision table:**

| Program policy | BBOT flags |
|---|---|
| Passive only | `-f passive` |
| No automated scanning | `-f passive` |
| Active allowed | `-f subdomain-enum web-basic safe` |
| Broad scope | `-f subdomain-enum web-thorough -ef deadly` |
| Full testing | `-p kitchen-sink -ef deadly` |

---

## Post-Scan Result Parsing

```bash
SCAN_DIR=~/bug_bounty/$COMPANY/bbot_scans/
LATEST=$(ls -td $SCAN_DIR/*/ | head -1)

# Summary
echo "Subdomains : $(cat $LATEST/subdomains.txt 2>/dev/null | wc -l)"
echo "URLs       : $(jq -r 'select(.type=="URL")' $LATEST/output.ndjson 2>/dev/null | wc -l)"
echo "Open ports : $(jq -r 'select(.type=="OPEN_TCP_PORT")' $LATEST/output.ndjson 2>/dev/null | wc -l)"
echo "Findings   : $(jq -r 'select(.type=="FINDING")' $LATEST/output.ndjson 2>/dev/null | wc -l)"
echo "Vulns      : $(jq -r 'select(.type=="VULNERABILITY")' $LATEST/output.ndjson 2>/dev/null | wc -l)"

# Critical/high vulns
jq -r 'select(.type=="VULNERABILITY") |
  select(.data.severity=="CRITICAL" or .data.severity=="HIGH") |
  "\(.data.severity): \(.data.name // .module) @ \(.data.host // .data.url // "")"' \
  $LATEST/output.ndjson 2>/dev/null | sort -u

# All findings
jq -r 'select(.type=="FINDING") | "\(.module): \(.data)"' \
  $LATEST/output.ndjson 2>/dev/null | sort -u

# Technologies
jq -r 'select(.type=="TECHNOLOGY") | .data' $LATEST/output.ndjson 2>/dev/null | sort -u

# Storage buckets
jq -r 'select(.type=="STORAGE_BUCKET") | .data' $LATEST/output.ndjson 2>/dev/null

# Event distribution
jq -r '.type' $LATEST/output.ndjson 2>/dev/null | sort | uniq -c | sort -rn
```

---

## Example User Requests

- "Run a passive BBOT scan for tesla.com"
- "Full engagement recon on hackerone.com"
- "Check subdomain takeover for *.acme.com"
- "Run nuclei against these 50 subdomains"
- "Hunt for S3 buckets for shopify"
- "Enumerate subdomains for apple.com passive only, no API keys"
- "Run a safe active scan with rate limit 10 req/sec"
- "Code leak hunt for github org microsoft"

---

**Version:** 1.0.0
**BBOT Version:** 2.8.2
**Last Updated:** 2026-02-26
