---
name: Safe Active Reconnaissance
description: Light active scanning — subdomain brute force, HTTP probing, port scan, and safe vuln checks
license: MIT
compatibility: MONK TAO skills system, Claude Code
metadata:
  author: shart123456
  usage: workflow file for MONK TAO skills system
  version: 0.2.0
  related files: "modules_dns_subdomain_enum.md, modules_port_scanning.md, modules_web_scanning.md, bbot_technical_reference.md"
  creation date: 2026-02-26
  last modified: 2026-02-26
---
# Workflow: Safe Active Reconnaissance

**Risk Level:** Low — direct contact with target but non-destructive
**Use When:** Program allows active scanning (most bug bounty programs)

---

## Overview

Safe active reconnaissance makes direct HTTP/TCP connections to the target to discover services, validate subdomains, probe web applications, and identify technologies. All modules in this workflow are non-destructive and won't trigger WAF blocks or create server errors.

**Modules Used:** httpx, sslcert, portscan, robots, securitytxt, social, oauth, ntlm, badsecrets, baddns, ironsight, wappalyzer, gowitness, dnsbrute, dnsbrute_mutations, dnscommonsrv, dnsbimi, dnscaa, dnstlsrpt, asn, cloudcheck, ipneighbor, ip2location, fingerprintx

---

## Phase 1: Subdomain Enumeration (Passive + Active Brute)

```bash
COMPANY="acme"
TARGET="acme.com"
OUT_OF_SCOPE="admin.acme.com staging.acme.com"

# Step 1: Passive discovery + active brute force + DNS record enumeration
bbot -t $TARGET \
     -f subdomain-enum \
     -m dnscommonsrv dnsbimi dnscaa dnstlsrpt \
     --blacklist $OUT_OF_SCOPE \
     -c dns.brute_threads=1000 \
        modules.dnsbrute.max_depth=3 \
     -om json subdomains \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_subdomain_enum
```

> **DNS record modules note:** `dnscommonsrv` (SRV records), `dnsbimi` (BIMI), `dnscaa` (CAA), and `dnstlsrpt` (TLS-RPT) are **active** modules — they send DNS queries to target nameservers. They are safe (non-destructive) but not passive.

---

## Phase 1B: IP and Network Intelligence

Understand the IP ownership and cloud infrastructure behind the target:

```bash
# ASN lookup + cloud tagging + IP neighbor discovery
bbot -t $TARGET \
     -m asn cloudcheck ipneighbor ip2location \
     --blacklist $OUT_OF_SCOPE \
     -om json \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_ip_intel
```

> **Scope warning for `asn`:** ASN lookup expands the target IP into a full CIDR range. If the target IP is hosted on AWS, Azure, or Cloudflare, this can produce thousands of IPs. Always review `IP_RANGE` events before proceeding and confirm the CIDR is owned by the target (check program scope).

**IP intelligence findings:**
```bash
LATEST=$(ls -td ~/bug_bounty/$COMPANY/bbot_scans/*/ | head -1)

# ASN info
jq -r 'select(.type=="ASN") | .data' $LATEST/output.ndjson

# IP ranges discovered
jq -r 'select(.type=="IP_RANGE") | .data' $LATEST/output.ndjson

# Cloud providers tagged
jq -r 'select(.module=="cloudcheck") | select(.type=="FINDING") | .data' $LATEST/output.ndjson

# IP neighbors (other hosts on same IPs)
jq -r 'select(.module=="ipneighbor") | select(.type=="DNS_NAME") | .data' $LATEST/output.ndjson
```

---

## Phase 2: Port + Web Discovery

```bash
# Step 2: Port scan + HTTP probe all discovered subdomains
bbot -t $TARGET \
     -m portscan fingerprintx sslcert httpx robots securitytxt social oauth \
        ironsight wappalyzer \
     --blacklist $OUT_OF_SCOPE \
     -c modules.portscan.top_ports=1000 \
        modules.portscan.rate=300 \
        modules.httpx.threads=50 \
     -om json csv subdomains \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_port_web
```

**`fingerprintx`:** Identifies service protocols on open ports (beyond HTTP/HTTPS). Detects SSH, FTP, SMTP, RDP, VNC, and 20+ other services without banner grabbing. Complements `portscan` and `sslcert`.

**`wappalyzer`:** Technology fingerprinting via Wappalyzer rules DB. Runs alongside `ironsight` for better tech stack coverage.

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
| `dnscommonsrv` | DNS Only | Very Low | Always — SRV records |
| `dnsbimi/caa/tlsrpt` | DNS Only | Very Low | Email infrastructure targets |
| `asn` | No | Low | When CIDR ownership confirmed |
| `cloudcheck` | No | Very Low | Always — cloud IP tagging |
| `ipneighbor` | No | Low | Shared hosting environments |
| `fingerprintx` | Yes | Low | Non-HTTP service identification |
| `wappalyzer` | Yes | Low | Always — tech detect (pair with ironsight) |

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

## Memory Integration

After completing safe active recon, store findings to Qdrant (if `qdrant-store` is available):

**At scan start — query prior intel:**
```
Tool: qdrant-find
Collection: target_intel
Query: "scope WAF rate limits technology stack [TARGET]"
```

**After scan — store vulnerability findings:**
For each VULNERABILITY or FINDING of severity medium+:
```
Tool: qdrant-store
Collection: scan_findings
Content: [severity] vulnerability [name] found on [host]:[port] via [module]
Metadata: {
  "target": "[TARGET]",
  "finding_type": "vulnerability",
  "data": "[vuln name and host]",
  "severity": "high",
  "source_module": "[module]",
  "host": "[host]",
  "port": [port],
  "scan_id": "[SCAN_NAME]",
  "scan_type": "safe_active",
  "timestamp": "[ISO 8601]"
}
```

**After scan — store technology stack to target_intel:**
```
Tool: qdrant-store
Collection: target_intel
Content: [TARGET] technology stack: [technologies] — detected during safe active recon
Metadata: {
  "target": "[TARGET]",
  "intel_type": "technology",
  "data": "[technologies]",
  "confidence": "verified",
  "source": "scan",
  "timestamp": "[ISO 8601]"
}
```

**After scan — store open ports summary:**
```
Tool: qdrant-store
Collection: scan_findings
Content: Safe active scan on [TARGET] found [N] open ports: [notable ports]
Metadata: {
  "target": "[TARGET]",
  "finding_type": "open_port",
  "data": "[port list]",
  "severity": "info",
  "source_module": "portscan",
  "scan_id": "[SCAN_NAME]",
  "scan_type": "safe_active",
  "timestamp": "[ISO 8601]"
}
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
