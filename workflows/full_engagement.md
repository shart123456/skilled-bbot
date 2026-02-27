---
name: Full Bug Bounty Engagement
description: Comprehensive 8-phase engagement covering passive intel through visual survey with all safe-active modules
license: MIT
compatibility: MONK TAO skills system, Claude Code
metadata:
  author: shart123456
  usage: workflow file for MONK TAO skills system
  version: 0.2.0
  related files: "bbot_technical_reference.md, modules_dns_subdomain_enum.md, modules_web_scanning.md, modules_vulnerability_scanning.md, modules_cloud_enumeration.md, modules_code_repository.md"
  creation date: 2026-02-26
  last modified: 2026-02-26
---
# Workflow: Full Bug Bounty Engagement

**Risk Level:** Medium-High (excludes deadly modules by default)
**Use When:** Comprehensive bug bounty engagement, full program scope, authorized testing

---

## Overview

The full engagement workflow combines all BBOT capabilities for maximum attack surface discovery. Runs in sequential phases to ensure each phase builds on previous results.

---

## Pre-Scan Checklist

```bash
# 1. Verify scope
cat ~/bug_bounty/$COMPANY/scope_analysis.md

# 2. Set up blacklist
cat ~/bug_bounty/$COMPANY/targets/out_of_scope.txt

# 3. Dry run to preview
bbot -t $TARGET \
     -f subdomain-enum web-basic cloud-enum code-enum \
     -ef deadly aggressive \
     --dry-run

# 4. Verify API keys
echo "GitHub: $GITHUB_TOKEN"
echo "Shodan: $SHODAN_KEY"
echo "Hunter: $HUNTER_KEY"
echo "VT: $VT_KEY"
```

---

## Phase 1: Intelligence Gathering (Passive, ~30-60 min)

```bash
COMPANY="acme"
TARGET="acme.com"
BLACKLIST="admin.acme.com staging.acme.com legacy.acme.com"

# Full passive intel
bbot -t $TARGET \
     -f subdomain-enum email-enum code-enum passive \
     --blacklist $BLACKLIST \
     -c modules.virustotal.api_key=$VT_KEY \
        modules.securitytrails.api_key=$ST_KEY \
        modules.chaos.api_key=$CHAOS_KEY \
        modules.shodan_dns.api_key=$SHODAN_KEY \
        modules.hunterio.api_key=$HUNTER_KEY \
        modules.github_org.api_key=$GITHUB_TOKEN \
        modules.github_codesearch.api_key=$GITHUB_TOKEN \
        modules.postman.api_key=$POSTMAN_KEY \
     -om json csv subdomains emails \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_phase1_passive
```

---

## Phase 2: Active Discovery (~1-3 hours)

```bash
# Subdomain brute + port scan + web probe
bbot -t $TARGET \
     -f subdomain-enum web-basic safe portscan \
     --blacklist $BLACKLIST \
     -c modules.portscan.top_ports=1000 \
        modules.portscan.rate=300 \
        dns.brute_threads=1000 \
        web_requests_per_second=20 \
        modules.dnsbrute.max_depth=5 \
     -om json csv subdomains \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_phase2_active
```

---

## Phase 3: Cloud Enumeration (~30-60 min)

```bash
bbot -t $TARGET \
     -p cloud-enum \
     --blacklist $BLACKLIST \
     -c modules.bucket_amazon.permutations=true \
        modules.bucket_google.permutations=true \
        modules.bucket_microsoft.permutations=true \
        modules.censys_ip.api_id=$CENSYS_ID \
        modules.censys_ip.api_secret=$CENSYS_SECRET \
     -om json web_report \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_phase3_cloud
```

---

## Phase 4: Web App Audit (~2-6 hours)

```bash
bbot -t $TARGET \
     -f web-basic web-thorough \
     -ef deadly \
     --blacklist $BLACKLIST \
     -c modules.httpx.store_responses=true \
        web_requests_per_second=10 \
     -om json web_report sqlite \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_phase4_web
```

---

## Phase 5: Vulnerability Scanning (~2-4 hours)

```bash
bbot -t $TARGET \
     -m httpx nuclei badsecrets retirejs wpscan \
        baddns baddns_zone ntlm dotnetnuke ajaxpro telerik \
     --blacklist $BLACKLIST \
     -c modules.nuclei.mode=technology \
        modules.nuclei.severity=critical,high,medium \
        modules.nuclei.ratelimit=50 \
        modules.wpscan.api_key=$WPSCAN_KEY \
     -om json web_report \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_phase5_vulns
```

---

## Phase 6: Credential & Code Secrets (~30-60 min)

```bash
bbot -t $TARGET \
     -m github_org github_codesearch postman postman_download \
        trufflehog git gitdumper \
     --blacklist $BLACKLIST \
     -c modules.github_org.api_key=$GITHUB_TOKEN \
        modules.github_org.include_member_repos=true \
        modules.github_codesearch.api_key=$GITHUB_TOKEN \
        modules.postman.api_key=$POSTMAN_KEY \
     -om json \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_phase6_secrets
```

---

## Phase 7: Subdomain Takeover Check (~30 min)

```bash
bbot -t $TARGET \
     -p baddns-intense \
     --blacklist $BLACKLIST \
     -c modules.baddns.only_high_confidence=false \
     -om json web_report \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_phase7_takeovers
```

---

## Phase 8: Visual Survey (Screenshots, ~1-2 hours)

```bash
bbot -t $TARGET \
     -m httpx gowitness social \
     --blacklist $BLACKLIST \
     -c modules.gowitness.resolution_x=1440 \
        modules.gowitness.resolution_y=900 \
     -om json \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_phase8_screenshots
```

---

## One-Shot Kitchen Sink (Excluding Deadly)

For when you want everything at once:

```bash
bbot -t $TARGET \
     -p kitchen-sink \
     -ef deadly \
     --blacklist $BLACKLIST \
     -c modules.portscan.top_ports=1000 \
        dns.brute_threads=1000 \
        web_requests_per_second=15 \
        modules.nuclei.mode=technology \
        modules.nuclei.severity=critical,high \
        modules.bucket_amazon.permutations=true \
        modules.bucket_google.permutations=true \
        modules.bucket_microsoft.permutations=true \
        modules.github_org.api_key=$GITHUB_TOKEN \
        modules.github_codesearch.api_key=$GITHUB_TOKEN \
        modules.hunterio.api_key=$HUNTER_KEY \
        modules.chaos.api_key=$CHAOS_KEY \
     -om json csv subdomains emails web_report sqlite \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_kitchen_sink
```

---

## Post-Scan Triage

```bash
COMPANY="acme"
SCANS_DIR=~/bug_bounty/$COMPANY/bbot_scans/

# Aggregate all findings across all phase scans
echo "=== CRITICAL/HIGH VULNERABILITIES ==="
for scandir in $SCANS_DIR/*/; do
  jq -r 'select(.type=="VULNERABILITY") |
    select(.data.severity=="CRITICAL" or .data.severity=="HIGH") |
    "\(.data.severity): \(.data.name // .module) @ \(.data.host // .data.url // .data)"' \
    "$scandir/output.ndjson" 2>/dev/null
done | sort -u

echo ""
echo "=== ALL FINDINGS ==="
for scandir in $SCANS_DIR/*/; do
  jq -r 'select(.type=="FINDING") | "\(.module): \(.data)"' \
    "$scandir/output.ndjson" 2>/dev/null
done | sort -u

echo ""
echo "=== SUBDOMAIN COUNT ==="
cat $SCANS_DIR/*/subdomains.txt 2>/dev/null | sort -u | wc -l

echo ""
echo "=== TOP BUG CLASSES ==="
for scandir in $SCANS_DIR/*/; do
  jq -r 'select(.type=="FINDING" or .type=="VULNERABILITY") | .module' \
    "$scandir/output.ndjson" 2>/dev/null
done | sort | uniq -c | sort -rn | head -20
```

---

## Priority Investigation Queue

After scan, prioritize investigation in this order:

1. **CRITICAL nuclei findings** — RCE, SQLi, auth bypass
2. **badsecrets findings** — Direct deserialization RCE potential
3. **Open storage buckets** — bucket_amazon/google/microsoft
4. **TruffleHog secrets** — Live API keys, tokens
5. **Subdomain takeovers** — baddns HIGH confidence
6. **Exposed .git directories** — Source code exposure
7. **HIGH nuclei findings** — CVEs, misconfigurations
8. **Postman secrets** — Hardcoded API keys in collections
9. **Vulnerable JS libraries** — retirejs findings
10. **Parameter reflections** — XSS candidates

---

## Engagement Time Estimates

| Phase | Modules | Duration |
|---|---|---|
| Phase 1: Passive Intel | crt, dnsdumpster, github_org, etc. | 30-60 min |
| Phase 2: Active Discovery | portscan, httpx, dnsbrute | 1-3 hrs |
| Phase 3: Cloud Enum | bucket_*, azure_* | 30-60 min |
| Phase 4: Web Audit | httpx, web-thorough preset | 2-6 hrs |
| Phase 5: Vuln Scan | nuclei, badsecrets, wpscan | 2-4 hrs |
| Phase 6: Code/Secrets | github, trufflehog, postman | 30-60 min |
| Phase 7: Takeovers | baddns-intense preset | 30 min |
| Phase 8: Screenshots | gowitness | 1-2 hrs |
| **Total** | | **8-18 hrs** |
