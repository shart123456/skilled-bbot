---
name: Passive Reconnaissance
description: Zero-contact reconnaissance using certificate transparency, passive DNS, and API sources only
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
# Workflow: Passive Reconnaissance

**Risk Level:** Zero — no contact with target infrastructure
**Use When:** Program requires passive-only, initial discovery, or unknown scope

---

## Overview

Passive reconnaissance uses only third-party data sources — certificate transparency logs, DNS databases, API services, search engines — to map a target's attack surface without ever touching the target directly.

**Modules Used:** anubisdb, bufferoverrun, crt, crt_db, dnsdumpster, hackertarget, myssl, otx, rapiddns, sitedossier, subdomaincenter, subdomainradar, urlscan, viewdns, wayback, certspotter, chaos, virustotal, securitytrails, shodan_dns, azure_realm, azure_tenant, emailformat, hunterio, skymem, pgp, github_org, postman

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
        urlscan viewdns wayback \
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

## Phase 5: Code Repository Intelligence

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

---

## Safety Checklist

- [ ] No modules with `active` flag included
- [ ] `--blacklist` not needed (passive sources filter themselves)
- [ ] Rate limits respected (API services auto-throttle)
- [ ] Output directory set outside public repos
- [ ] API keys not hardcoded in scripts
