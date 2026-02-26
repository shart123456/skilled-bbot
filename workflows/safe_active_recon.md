---
name: Safe Active Reconnaissance
description: Light active scanning — subdomain brute force, HTTP probing, port scan, and safe vuln checks
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
# Workflow: Safe Active Reconnaissance

**Risk Level:** Low — direct contact with target but non-destructive
**Use When:** Program allows active scanning (most bug bounty programs)

---

## Overview

Safe active reconnaissance makes direct HTTP/TCP connections to the target to discover services, validate subdomains, probe web applications, and identify technologies. All modules in this workflow are non-destructive and won't trigger WAF blocks or create server errors.

**Modules Used:** httpx, sslcert, portscan, robots, securitytxt, social, oauth, ntlm, badsecrets, baddns, ironsight, gowitness, dnsbrute, dnsbrute_mutations

---

## Phase 1: Subdomain Enumeration (Passive + Active Brute)

```bash
COMPANY="acme"
TARGET="acme.com"
OUT_OF_SCOPE="admin.acme.com staging.acme.com"

# Step 1: Passive discovery + active brute force
bbot -t $TARGET \
     -f subdomain-enum \
     --blacklist $OUT_OF_SCOPE \
     -c dns.brute_threads=1000 \
        modules.dnsbrute.max_depth=3 \
     -om json subdomains \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_subdomain_enum
```

---

## Phase 2: Port + Web Discovery

```bash
# Step 2: Port scan + HTTP probe all discovered subdomains
bbot -t $TARGET \
     -m portscan sslcert httpx robots securitytxt social oauth \
     --blacklist $OUT_OF_SCOPE \
     -c modules.portscan.top_ports=1000 \
        modules.portscan.rate=300 \
        modules.httpx.threads=50 \
     -om json csv subdomains \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_port_web
```

---

## Phase 3: Initial Vulnerability Surface (Safe Only)

```bash
# Step 3: Safe vuln checks — no aggressive/deadly modules
bbot -t $TARGET \
     -m httpx badsecrets baddns ntlm ironsight sslcert \
     --blacklist $OUT_OF_SCOPE \
     -om json web_report \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_safe_vuln
```

---

## Phase 4: Visual Survey

```bash
# Step 4: Screenshot everything for manual review
bbot -t $TARGET \
     -m httpx gowitness \
     --blacklist $OUT_OF_SCOPE \
     -c modules.gowitness.resolution_x=1440 \
        modules.gowitness.resolution_y=900 \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_screenshots
```

---

## One-Shot Safe Active Command

Combines all phases into a single scan with appropriate flag filters:

```bash
COMPANY="acme"
TARGET="acme.com"

bbot -t $TARGET \
     -f subdomain-enum web-basic safe \
     --blacklist admin.acme.com legacy.acme.com \
     -c modules.portscan.top_ports=1000 \
        modules.portscan.rate=300 \
        dns.brute_threads=1000 \
        web_requests_per_second=10 \
     -om json csv subdomains web_report \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_safe_active
```

---

## Bug Bounty Program Rate Limiting

Respect program-specified rate limits:

```bash
# 10 requests/second limit
bbot -t $TARGET \
     -f subdomain-enum web-basic safe \
     -c web_requests_per_second=10 \
        modules.portscan.rate=100 \
     -n slow_respectful_scan

# Strict 1 req/sec (very conservative programs)
bbot -t $TARGET \
     -f subdomain-enum web-basic safe \
     -c web_requests_per_second=1 \
        http_timeout=30 \
     -n ultra_conservative_scan
```

---

## Safe Active Module Selection Guide

| Module | Touched Target? | Noise Level | Use When |
|---|---|---|---|
| `httpx` | Yes | Low | Always — core web probe |
| `sslcert` | Yes | Very Low | Always — cert extraction |
| `portscan` | Yes | Medium | Active scanning allowed |
| `robots` | Yes | Very Low | Always — passive-ish |
| `securitytxt` | Yes | Very Low | Always — passive-ish |
| `social` | Yes | Very Low | Always |
| `oauth` | Yes | Low | Always |
| `ntlm` | Yes | Low | .NET/Windows targets |
| `badsecrets` | Yes | Low | ASP.NET/framework targets |
| `baddns` | Yes | Low | Always — takeover check |
| `ironsight` | Yes | Low | Always — tech detect |
| `gowitness` | Yes | Low | Visual survey |
| `dnsbrute` | DNS Only | Medium | Always |

---

## Result Analysis After Safe Active Scan

```bash
COMPANY="acme"
SCAN_DIR=~/bug_bounty/$COMPANY/bbot_scans/
LATEST=$(ls -td $SCAN_DIR/*/ | head -1)

# Summary stats
echo "=== Scan Summary ==="
echo "Subdomains: $(cat $LATEST/subdomains.txt 2>/dev/null | wc -l)"
echo "URLs: $(jq -r 'select(.type=="URL") | .data' $LATEST/output.ndjson 2>/dev/null | wc -l)"
echo "Open Ports: $(jq -r 'select(.type=="OPEN_TCP_PORT") | .data' $LATEST/output.ndjson 2>/dev/null | wc -l)"
echo "Findings: $(jq -r 'select(.type=="FINDING") | .data' $LATEST/output.ndjson 2>/dev/null | wc -l)"
echo "Technologies: $(jq -r 'select(.type=="TECHNOLOGY") | .data' $LATEST/output.ndjson 2>/dev/null | sort -u | wc -l)"

# Quick wins to investigate
echo ""
echo "=== Interesting Findings ==="
jq -r 'select(.type=="FINDING") | "\(.module): \(.data)"' $LATEST/output.ndjson 2>/dev/null

# Discovered technologies
echo ""
echo "=== Technologies Detected ==="
jq -r 'select(.type=="TECHNOLOGY") | .data' $LATEST/output.ndjson 2>/dev/null | sort -u

# Open ports summary
echo ""
echo "=== Open Ports ==="
jq -r 'select(.type=="OPEN_TCP_PORT") | .data' $LATEST/output.ndjson 2>/dev/null | sort -u
```

---

## Next Steps After Safe Active

Based on findings, proceed to targeted testing:

```bash
# If WordPress found → WordPress audit
bbot -t wp.acme.com -m httpx wpscan retirejs \
     -c modules.wpscan.api_key=$WPSCAN_KEY

# If IIS found → IIS shortname test
bbot -t iis.acme.com -m httpx iis_shortnames ffuf_shortnames

# If admin panels found → nuclei tech-specific scan
bbot -t admin.acme.com -m httpx nuclei \
     -c modules.nuclei.mode=technology

# If GraphQL found → introspection
bbot -t api.acme.com -m httpx graphql_introspection
```
