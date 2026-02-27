---
name: Passive Reconnaissance
description: Zero-contact reconnaissance using certificate transparency, passive DNS, and API sources only
license: MIT
compatibility: MONK TAO skills system, Claude Code
metadata:
  author: shart123456
  usage: workflow file for MONK TAO skills system
  version: 0.2.0
  related files: "modules_dns_subdomain_enum.md, modules_email_credentials.md, modules_ip_intelligence.md, bbot_technical_reference.md"
  creation date: 2026-02-26
  last modified: 2026-02-26
---
# Workflow: Passive Reconnaissance

**Risk Level:** Zero — no contact with target infrastructure
**Use When:** Program requires passive-only, initial discovery, or unknown scope

---

## Overview

Passive reconnaissance uses only third-party data sources — certificate transparency logs, DNS databases, API services, search engines — to map a target's attack surface without ever touching the target directly.

**Modules Used:** anubisdb, bufferoverrun, crt, crt_db, dnsdumpster, hackertarget, myssl, otx, rapiddns, sitedossier, subdomaincenter, subdomainradar, urlscan, viewdns, wayback, certspotter, chaos, virustotal, securitytrails, shodan_dns, azure_realm, azure_tenant, emailformat, hunterio, skymem, pgp, github_org, postman, affiliates, zoomeye, c99, fullhunt, leakix, passivetotal, bevigil, trickest, digitorus

---

## Phase 1: Minimum Viable Passive (No API Keys)

These sources require zero authentication:

```bash
COMPANY="acme"
TARGET="acme.com"

bbot -t $TARGET \
     -m anubisdb bufferoverrun crt crt_db \
        dnsdumpster hackertarget myssl otx \
        rapiddns sitedossier subdomaincenter \
        urlscan viewdns wayback affiliates \
     -om json subdomains emails \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_passive_nokey
```

**Expected Output:**
- Hundreds to thousands of subdomains from cert logs
- Historical URLs from Wayback Machine
- Subdomain data from passive DNS sources

---

## Phase 2: Full Passive (API Keys Recommended)

```bash
COMPANY="acme"
TARGET="acme.com"

bbot -t $TARGET \
     -f subdomain-enum passive safe \
     -c modules.virustotal.api_key=$VT_KEY \
        modules.securitytrails.api_key=$ST_KEY \
        modules.certspotter.api_key=$CS_KEY \
        modules.chaos.api_key=$CHAOS_KEY \
        modules.otx.api_key=$OTX_KEY \
        modules.shodan_dns.api_key=$SHODAN_KEY \
        modules.hunterio.api_key=$HUNTER_KEY \
        modules.github_org.api_key=$GITHUB_TOKEN \
        modules.github_codesearch.api_key=$GITHUB_TOKEN \
        modules.postman.api_key=$POSTMAN_KEY \
     -om json csv subdomains emails web_report \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_passive_full
```

---

## Phase 3: Cloud Intelligence (Passive)

```bash
bbot -t $TARGET \
     -m azure_realm azure_tenant \
        shodan_idb \
     -c modules.censys_dns.api_id=$CENSYS_ID \
        modules.censys_dns.api_secret=$CENSYS_SECRET \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_passive_cloud
```

---

## Phase 4: Email & People Intelligence

```bash
bbot -t $TARGET \
     -p email-enum \
     -c modules.hunterio.api_key=$HUNTER_KEY \
     -om json emails \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_email_enum
```

---

## Phase 5: Paid Intelligence Sources

Unlock maximum coverage with paid API keys:

```bash
bbot -t $TARGET \
     -m c99 fullhunt leakix zoomeye passivetotal \
        bevigil trickest digitorus affiliates \
     -c modules.c99.api_key=$C99_KEY \
        modules.fullhunt.api_key=$FULLHUNT_KEY \
        modules.leakix.api_key=$LEAKIX_KEY \
        modules.zoomeye.api_key=$ZOOMEYE_KEY \
        modules.passivetotal.api_key=$PT_KEY \
        modules.passivetotal.username=$PT_USER \
        modules.bevigil.api_key=$BEVIGIL_KEY \
        modules.trickest.api_key=$TRICKEST_KEY \
     -om json subdomains \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_paid_intel
```

---

## Phase 6: Code Repository Intelligence

```bash
bbot -t $TARGET \
     -p code-enum \
     -c modules.github_org.api_key=$GITHUB_TOKEN \
        modules.github_org.include_member_repos=true \
        modules.github_codesearch.api_key=$GITHUB_TOKEN \
        modules.postman.api_key=$POSTMAN_KEY \
     -om json \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_code_enum
```

---

## Consolidated All-Passive Command

```bash
bbot -t $TARGET \
     -f subdomain-enum email-enum code-enum passive safe \
     -c modules.virustotal.api_key=$VT_KEY \
        modules.securitytrails.api_key=$ST_KEY \
        modules.chaos.api_key=$CHAOS_KEY \
        modules.hunterio.api_key=$HUNTER_KEY \
        modules.github_org.api_key=$GITHUB_TOKEN \
        modules.github_codesearch.api_key=$GITHUB_TOKEN \
        modules.postman.api_key=$POSTMAN_KEY \
     -om json csv subdomains emails web_report \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_full_passive
```

---

## Result Analysis

### Extract and Deduplicate Subdomains
```bash
SCAN_DIR=~/bug_bounty/$COMPANY/bbot_scans/
LATEST=$(ls -td $SCAN_DIR/*/ | head -1)

# All unique subdomains
cat $LATEST/subdomains.txt | sort -u | wc -l
cat $LATEST/subdomains.txt | sort -u > /tmp/subdomains_unique.txt

# Query JSON for specific types
cat $LATEST/output.ndjson | jq -r 'select(.type=="DNS_NAME") | .data' | sort -u

# View findings
cat $LATEST/output.ndjson | jq 'select(.type=="FINDING")'

# View code repositories discovered
cat $LATEST/output.ndjson | jq 'select(.type=="CODE_REPOSITORY") | .data'
```

### Quick Stats
```bash
# Event type distribution
cat $LATEST/output.ndjson | jq -r '.type' | sort | uniq -c | sort -rn

# Module contribution to subdomains
cat $LATEST/output.ndjson | \
  jq -r 'select(.type=="DNS_NAME") | .module' | \
  sort | uniq -c | sort -rn
```

---

## Passive Scan Decision Matrix

| Source | Free | API Key Needed | Value |
|---|---|---|---|
| crt.sh | Yes | No | High |
| anubisdb | Yes | No | Medium |
| dnsdumpster | Yes | No | Medium |
| wayback | Yes | No | High |
| hackertarget | Yes | No | Medium |
| virustotal | Partial | Yes (free) | High |
| securitytrails | No | Yes (paid) | Very High |
| shodan_dns | No | Yes (paid) | High |
| chaos | No | Yes (free) | High |
| hunterio | Partial | Yes (free) | High |
| github_codesearch | No | Yes (free) | Very High |
| postman | No | Yes (free) | High |
| affiliates | Yes | No | Medium |
| zoomeye | No | Yes (paid) | High (APAC) |
| c99 | No | Yes (paid) | High |
| fullhunt | Partial | Yes (free) | High |
| leakix | Partial | Yes (free) | High |
| passivetotal | No | Yes (paid) | Very High |
| bevigil | Partial | Yes (free) | Medium (mobile) |
| trickest | No | Yes (paid) | High |
| digitorus | Yes | No | Medium |

---

## Memory Integration

After completing passive recon, store key findings to Qdrant (if `qdrant-store` is available):

```bash
# Get summary stats for storage
SCAN_DIR=~/bug_bounty/$COMPANY/bbot_scans/
LATEST=$(ls -td $SCAN_DIR/*/ | head -1)
SUBDOMAIN_COUNT=$(cat $LATEST/subdomains.txt 2>/dev/null | wc -l)
FINDING_COUNT=$(jq -r 'select(.type=="FINDING")' $LATEST/output.ndjson 2>/dev/null | wc -l)
```

**Store scan summary (scan_findings):**
```
Tool: qdrant-store
Collection: scan_findings
Content: Passive recon on [TARGET] found [N] subdomains, [N] findings from cert transparency and passive DNS sources
Metadata: {
  "target": "[TARGET]",
  "finding_type": "subdomain",
  "data": "[N] subdomains discovered",
  "severity": "info",
  "source_module": "passive_recon_workflow",
  "scan_id": "[SCAN_NAME]",
  "scan_type": "passive",
  "timestamp": "[ISO 8601]"
}
```

**Store each notable finding individually** (vulnerabilities, code repos, storage buckets):
```
Tool: qdrant-store
Collection: scan_findings
Content: [natural language description]
Metadata: { "target": "...", "finding_type": "...", "data": "...", "severity": "...", ... }
```

**At scan start — query prior intel:**
```
Tool: qdrant-find
Collection: target_intel
Query: "scope and technology for [TARGET]"
```

---

## Safety Checklist

1. Confirm no modules with `active` flag are included
2. Confirm `--blacklist` not needed (passive sources filter themselves)
3. Confirm rate limits respected (API services auto-throttle)
4. Confirm output directory is set outside public repos
5. Confirm API keys are not hardcoded in scripts
