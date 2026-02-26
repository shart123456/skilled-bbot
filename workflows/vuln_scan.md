---
name: Vulnerability Scanning
description: Nuclei, badsecrets, retirejs, and baddns against live hosts for focused vulnerability detection
license: MIT
compatibility: MONK TAO skills system, Claude Code
metadata:
  author: shart123456
  usage: workflow file for MONK TAO skills system
  version: 0.1.0
  related files: ""
  creation date: 2026-02-26
  last modified: 2026-02-26
---
# Workflow: Vulnerability Scanning

**Risk Level:** Medium to High (nuclei/ffuf are aggressive)
**Use When:** Program allows active vulnerability scanning, full pentest scope

---

## Overview

Vulnerability scanning combines technology-aware nuclei templates, framework-specific modules, JavaScript library analysis, and known secret detection to identify exploitable vulnerabilities at scale.

**Modules Used:** nuclei, badsecrets, retirejs, wpscan, dotnetnuke, ajaxpro, telerik, ntlm, aspnet_bin_exposure, baddns, graphql_introspection, httpx

---

## Phase 1: Technology Detection First

Always detect technologies before running vuln modules — enables targeted testing:

```bash
COMPANY="acme"
TARGET="acme.com"

# Step 1: Identify tech stack
bbot -t $TARGET \
     -m httpx ironsight builtwith \
     -c modules.builtwith.api_key=$BUILTWITH_KEY \
     -om json \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_tech_detect

# Review technologies
jq -r 'select(.type=="TECHNOLOGY") | .data' \
   ~/bug_bounty/$COMPANY/bbot_scans/*/output.ndjson | sort -u
```

---

## Phase 2: Nuclei - Technology-Aware Mode

```bash
# Technology-aware: auto-selects templates based on what ironsight/httpx found
bbot -t $TARGET \
     -m httpx nuclei \
     -c modules.nuclei.mode=technology \
        modules.nuclei.ratelimit=50 \
        modules.nuclei.concurrency=10 \
     -om json web_report \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_nuclei_tech
```

---

## Phase 3: Nuclei - Critical/High CVEs

```bash
# Focus on highest severity only
bbot -t $TARGET \
     -m httpx nuclei \
     -c modules.nuclei.mode=severe \
        modules.nuclei.severity=critical,high \
        modules.nuclei.ratelimit=100 \
        modules.nuclei.concurrency=25 \
     -om json web_report \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_nuclei_crit
```

---

## Phase 4: Known Secrets & Framework Vulnerabilities

```bash
# badsecrets: framework-specific known secret detection
bbot -t $TARGET \
     -m httpx badsecrets ntlm aspnet_bin_exposure \
     -om json \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_secrets_check
```

---

## Phase 5: JavaScript Library Vulnerabilities

```bash
# retirejs: find vulnerable JS libraries in use
bbot -t $TARGET \
     -m httpx retirejs \
     -om json web_report \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_retirejs
```

---

## Phase 6: Platform-Specific Scans

### WordPress Sites
```bash
bbot -t $TARGET \
     -m httpx wpscan retirejs badsecrets \
     -c modules.wpscan.api_key=$WPSCAN_KEY \
        modules.wpscan.enumerate="ap,at,u,cb,dbe" \
     -om json \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_wordpress
```

### .NET / IIS Sites
```bash
bbot -t $TARGET \
     -m httpx badsecrets dotnetnuke ajaxpro telerik \
        ntlm aspnet_bin_exposure iis_shortnames \
     -om json \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_dotnet
```

### GraphQL APIs
```bash
bbot -t $TARGET \
     -m httpx graphql_introspection nuclei \
     -c modules.nuclei.tags=graphql \
     -om json \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_graphql
```

---

## Nuclei Template Selection Guide

```bash
# By tags (most common)
bbot -t $TARGET -m httpx nuclei \
     -c modules.nuclei.tags=cve,rce,sqli

# By specific template path
bbot -t $TARGET -m httpx nuclei \
     -c modules.nuclei.templates=/opt/nuclei-templates/cves/2024/

# By severity
bbot -t $TARGET -m httpx nuclei \
     -c modules.nuclei.severity=critical,high

# Exclude noisy templates
bbot -t $TARGET -m httpx nuclei \
     -c modules.nuclei.mode=severe \
        modules.nuclei.etags=info,fuzzing

# Budget mode (top 5 most critical templates)
bbot -t $TARGET -m httpx nuclei \
     -c modules.nuclei.mode=budget \
        modules.nuclei.budget=5
```

**High-Value Template Tags:**
| Tag | What it Finds | Priority |
|---|---|---|
| `cve` | Known CVE exploits | Critical |
| `rce` | Remote code execution | Critical |
| `sqli` | SQL injection | High |
| `ssrf` | Server-side request forgery | High |
| `exposure` | File/data exposure | High |
| `default-login` | Default credentials | High |
| `takeover` | Service takeovers | High |
| `misconfig` | Misconfiguration | Medium |
| `xss` | Cross-site scripting | Medium |
| `lfi` | Local file inclusion | Medium |
| `tech` | Technology detection | Info |

---

## Rate-Limited Respectful Scanning

```bash
# Conservative (10 req/sec, 3 concurrent templates)
bbot -t $TARGET -m httpx nuclei \
     -c modules.nuclei.ratelimit=10 \
        modules.nuclei.concurrency=3 \
        modules.nuclei.mode=severe

# Standard (50 req/sec, 10 concurrent)
bbot -t $TARGET -m httpx nuclei \
     -c modules.nuclei.ratelimit=50 \
        modules.nuclei.concurrency=10 \
        modules.nuclei.mode=technology

# Fast (150 req/sec, 25 concurrent)
bbot -t $TARGET -m httpx nuclei \
     -c modules.nuclei.ratelimit=150 \
        modules.nuclei.concurrency=25 \
        modules.nuclei.severity=critical,high
```

---

## Comprehensive Vuln Scan (No Deadly)

```bash
# All vuln modules except deadly (ffuf, medusa)
bbot -t $TARGET \
     -m httpx badsecrets ntlm retirejs baddns \
        dotnetnuke ajaxpro telerik aspnet_bin_exposure \
        graphql_introspection nuclei wpscan \
     -c modules.nuclei.mode=technology \
        modules.nuclei.severity=critical,high,medium \
        modules.wpscan.api_key=$WPSCAN_KEY \
     -om json web_report sqlite \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_comprehensive_vuln
```

---

## Analyzing Vulnerability Results

```bash
LATEST=$(ls -td ~/bug_bounty/$COMPANY/bbot_scans/*/ | head -1)

# All vulnerabilities by severity
jq -r 'select(.type=="VULNERABILITY") |
  "\(.data.severity // "UNKNOWN"): \(.data.host // .data.url) - \(.data.name // .module)"' \
  $LATEST/output.ndjson | sort

# Critical only
jq 'select(.type=="VULNERABILITY") |
  select(.data.severity=="CRITICAL" or .data.severity=="critical")' \
  $LATEST/output.ndjson

# badsecrets findings (known secrets)
jq 'select(.module=="badsecrets") | select(.type=="FINDING" or .type=="VULNERABILITY")' \
   $LATEST/output.ndjson

# Nuclei findings
jq -r 'select(.module=="nuclei") |
  select(.type=="VULNERABILITY" or .type=="FINDING") |
  "\(.data.severity): \(.data.name) @ \(.data.host)"' \
  $LATEST/output.ndjson | sort

# Vulnerable JS libraries
jq 'select(.module=="retirejs") | select(.type=="VULNERABILITY")' \
   $LATEST/output.ndjson
```
