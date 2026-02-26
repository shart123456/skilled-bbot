---
name: Web Application Audit
description: Parameter mining, secret detection, web fuzzing, SSRF checks, and WAF fingerprinting
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
# Workflow: Web Application Audit

**Risk Level:** Medium — active web probing, parameter mining, content discovery
**Use When:** Deep web application testing, bug bounty web scope, API surface mapping

---

## Overview

The web application audit workflow provides thorough coverage of web attack surfaces — from basic HTTP probing through content discovery, parameter mining, technology fingerprinting, WAF detection, and API enumeration. Excludes deadly fuzzing modules by default to stay safe for most programs.

**Modules Used:** httpx, robots, securitytxt, social, oauth, graphql_introspection, newsletters, gowitness, wafw00f, ironsight, ntlm, host_header, url_manipulation, paramminer_getparams, paramminer_headers, paramminer_cookies, reflected_parameters, bypass403, vhost, code_repository, filedownload, extractous, smuggler

---

## Phase 1: Surface Mapping (Safe — Always Run)

Establishes baseline web presence, discovers all accessible endpoints:

```bash
COMPANY="acme"
TARGET="acme.com"
BLACKLIST="admin.acme.com staging.acme.com"

bbot -t $TARGET \
     -m httpx robots securitytxt social oauth \
        graphql_introspection newsletters code_repository \
     --blacklist $BLACKLIST \
     -c modules.httpx.threads=50 \
        modules.httpx.store_responses=true \
        web_requests_per_second=20 \
     -om json csv subdomains \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_web_surface
```

**Discovers:**
- All live web services (URL, HTTP_RESPONSE events)
- robots.txt paths — hidden directories, admin panels
- security.txt — contact info, bug bounty platform, PGP keys
- Social media profiles — LinkedIn, GitHub, Twitter
- OAuth/OpenID endpoints — auth attack surface
- GraphQL endpoints with introspection enabled
- Links back to code repositories

---

## Phase 2: Technology Identification

Know the stack before testing it:

```bash
bbot -t $TARGET \
     -m httpx ironsight wafw00f ntlm \
     --blacklist $BLACKLIST \
     -c modules.builtwith.api_key=$BUILTWITH_KEY \
     -om json \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_web_tech
```

**Discovers:**
- Web frameworks (Django, Rails, ASP.NET, Laravel, etc.)
- WAF vendor and type (Cloudflare, AWS WAF, Imperva, etc.)
- NTLM authentication — reveals internal Windows domain names
- CMS platforms (WordPress, Drupal, Joomla)
- JavaScript libraries and versions

**Why WAF Detection Matters First:**
Understanding the WAF before running paramminer or ffuf lets you select bypass techniques specific to that vendor.

---

## Phase 3: Visual Survey (Screenshots)

Visual review of all discovered endpoints before deep testing:

```bash
bbot -t $TARGET \
     -m httpx gowitness \
     --blacklist $BLACKLIST \
     -c modules.gowitness.resolution_x=1440 \
        modules.gowitness.resolution_y=900 \
        modules.gowitness.timeout=15 \
     -om json \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_web_screenshots
```

**Use screenshots to:**
- Identify admin panels, login pages, dashboards
- Spot unusual or unexpected applications
- Find staging/dev environments
- Prioritize which apps to test manually

---

## Phase 4: Content Discovery (Directory Brute-Force)

**Warning:** ffuf is flagged `deadly` — confirm program allows active content discovery.

```bash
# Only if program allows web fuzzing
bbot -t $TARGET \
     -m httpx ffuf \
     --blacklist $BLACKLIST \
     -c modules.ffuf.wordlist=/opt/SecLists/Discovery/Web-Content/raft-medium-directories.txt \
        modules.ffuf.lines=10000 \
        modules.ffuf.rate=30 \
        modules.ffuf.extensions="php,asp,aspx,jsp,html,txt,bak,old,json,xml,yml,yaml" \
        web_requests_per_second=30 \
     -om json \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_web_fuzz
```

**Wordlist Recommendations by Target:**
| Target Type | Wordlist |
|---|---|
| General | `raft-medium-directories.txt` |
| IIS / .NET | `IIS.fuzz.txt` |
| API | `api-endpoints.txt` |
| PHP | `PHP.fuzz.txt` |
| Large scope | `directory-list-2.3-big.txt` |

---

## Phase 5: IIS Shortname Detection

If IIS detected (Windows/ASP.NET targets):

```bash
bbot -t $TARGET \
     -m httpx iis_shortnames ffuf_shortnames \
     --blacklist $BLACKLIST \
     -om json \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_iis_shortnames
```

---

## Phase 6: Virtual Host Discovery

Find hidden apps served on the same IP with different Host headers:

```bash
bbot -t $TARGET \
     -m httpx vhost \
     --blacklist $BLACKLIST \
     -c modules.vhost.wordlist=/opt/SecLists/Discovery/DNS/subdomains-top1million-5000.txt \
        modules.vhost.concurrency=10 \
     -om json \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_vhost
```

---

## Phase 7: Parameter Mining

Discover hidden GET parameters, headers, and cookies:

```bash
# Only if program allows active parameter testing
bbot -t $TARGET \
     -f web-paramminer \
     --blacklist $BLACKLIST \
     -c web_requests_per_second=10 \
     -om json \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_paramminer
```

**Runs:** `paramminer_getparams`, `paramminer_headers`, `paramminer_cookies`

**Then test reflected parameters:**
```bash
bbot -t $TARGET \
     -m httpx paramminer_getparams reflected_parameters \
     --blacklist $BLACKLIST \
     -c web_requests_per_second=5 \
     -om json \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_reflected_params
```

---

## Phase 7B: Lightweight Parameter Fuzzing

Test discovered parameters for common injection vulnerabilities:

```bash
# Only if program allows active vulnerability testing
# lightfuzz is flagged "aggressive"
bbot -t $TARGET \
     -m httpx paramminer_getparams lightfuzz \
     --blacklist $BLACKLIST \
     -c web_requests_per_second=5 \
        modules.lightfuzz.severity=MEDIUM \
     -om json \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_lightfuzz
```

**`lightfuzz` tests:** SQLi, XSS reflection, path traversal, SSTI, command injection

```bash
# Analyze lightfuzz findings
LATEST=$(ls -td ~/bug_bounty/$COMPANY/bbot_scans/*/ | head -1)

echo "=== LightFuzz Findings ==="
jq 'select(.module=="lightfuzz") | select(.type=="FINDING" or .type=="VULNERABILITY")' \
   $LATEST/output.ndjson

# Separate by severity
jq -r 'select(.module=="lightfuzz") |
  select(.type=="VULNERABILITY") |
  "\(.data.severity // "UNKNOWN"): \(.data.name // .data) @ \(.data.url // .data.host // "")"' \
  $LATEST/output.ndjson | sort -u
```

---

## Phase 8: Authentication & Injection Checks

```bash
bbot -t $TARGET \
     -m httpx host_header url_manipulation bypass403 \
     --blacklist $BLACKLIST \
     -om json \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_auth_injection
```

**Modules:**
- `host_header` — Host header injection (password reset poisoning, cache poisoning)
- `url_manipulation` — Path traversal, URL normalization bypasses
- `bypass403` — Bypass 403 Forbidden responses (IP spoofing headers, path tricks)

---

## Phase 9: HTTP Request Smuggling

```bash
# Only if program allows aggressive web testing
bbot -t $TARGET \
     -m httpx smuggler \
     --blacklist $BLACKLIST \
     -om json \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_smuggling
```

---

## Phase 10: File Download & Analysis

Grab documents exposed on the web surface:

```bash
bbot -t $TARGET \
     -m httpx filedownload extractous \
     --blacklist $BLACKLIST \
     -c modules.filedownload.extensions="pdf,docx,xlsx,txt,csv,xml,json,env,yml,yaml,bak,old,sql" \
        modules.filedownload.max_size=52428800 \
        modules.filedownload.output_folder=/opt/downloads/$COMPANY/ \
     -om json \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_file_download
```

---

## One-Shot Web-Thorough Preset

BBOT's built-in thorough web preset — good default for most programs:

```bash
bbot -t $TARGET -p web-thorough \
     -ef deadly \
     --blacklist $BLACKLIST \
     -c web_requests_per_second=15 \
        modules.httpx.store_responses=true \
     -om json web_report sqlite subdomains \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_web_thorough
```

---

## Full Web App Audit (All Phases, Excluding Deadly)

```bash
COMPANY="acme"
TARGET="acme.com"
BLACKLIST="admin.acme.com legacy.acme.com"

bbot -t $TARGET \
     -m httpx robots securitytxt social oauth \
        graphql_introspection newsletters code_repository \
        ironsight wafw00f ntlm \
        gowitness \
        host_header url_manipulation bypass403 \
        paramminer_getparams paramminer_headers paramminer_cookies \
        reflected_parameters lightfuzz \
        iis_shortnames vhost \
        filedownload extractous \
     --blacklist $BLACKLIST \
     -c modules.httpx.threads=50 \
        modules.httpx.store_responses=true \
        web_requests_per_second=15 \
        modules.gowitness.resolution_x=1440 \
        modules.gowitness.resolution_y=900 \
        modules.filedownload.extensions="pdf,docx,xlsx,txt,bak,sql,env,yml" \
     -om json csv web_report sqlite \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_full_web_audit
```

---

## API-Focused Web Audit

When target is primarily an API:

```bash
bbot -t $TARGET \
     -m httpx graphql_introspection oauth \
        paramminer_getparams paramminer_headers \
        reflected_parameters generic_ssrf \
        robots securitytxt \
     --blacklist $BLACKLIST \
     -c web_requests_per_second=10 \
        modules.httpx.store_responses=true \
     -om json web_report \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_api_audit
```

---

## Result Analysis

```bash
LATEST=$(ls -td ~/bug_bounty/$COMPANY/bbot_scans/*/ | head -1)

# Discovered URLs
echo "=== URLs Discovered ==="
jq -r 'select(.type=="URL") | .data' $LATEST/output.ndjson | sort -u | wc -l
jq -r 'select(.type=="URL") | .data' $LATEST/output.ndjson | sort -u

# Interesting paths from robots.txt
echo ""
echo "=== robots.txt Paths ==="
jq -r 'select(.module=="robots") | select(.type=="URL_UNVERIFIED") | .data' \
   $LATEST/output.ndjson | sort -u

# GraphQL findings
echo ""
echo "=== GraphQL Introspection ==="
jq 'select(.module=="graphql_introspection") | select(.type=="FINDING")' \
   $LATEST/output.ndjson

# WAF detected
echo ""
echo "=== WAF Detection ==="
jq -r 'select(.type=="WAF") | .data' $LATEST/output.ndjson

# NTLM domain disclosure
echo ""
echo "=== NTLM Domain Info ==="
jq 'select(.module=="ntlm") | select(.type=="FINDING")' $LATEST/output.ndjson

# Discovered parameters
echo ""
echo "=== Web Parameters Found ==="
jq -r 'select(.type=="WEB_PARAMETER") | .data' $LATEST/output.ndjson | sort -u

# Reflected parameters (XSS candidates)
echo ""
echo "=== Reflected Parameters (XSS Candidates) ==="
jq 'select(.module=="reflected_parameters") | select(.type=="FINDING")' \
   $LATEST/output.ndjson

# 403 bypasses
echo ""
echo "=== 403 Bypasses ==="
jq 'select(.module=="bypass403") | select(.type=="FINDING")' $LATEST/output.ndjson

# Host header injection
echo ""
echo "=== Host Header Findings ==="
jq 'select(.module=="host_header") | select(.type=="FINDING")' $LATEST/output.ndjson

# Downloaded files
echo ""
echo "=== Files Downloaded ==="
jq -r 'select(.module=="filedownload") | select(.type=="FILESYSTEM") | .data' \
   $LATEST/output.ndjson
```

---

## Web Audit Priority Queue

After running, investigate in this order:

1. **GraphQL introspection enabled** — Full schema reveals all mutations/queries
2. **403 bypasses found** — Potentially unauthorized access to restricted resources
3. **Reflected parameters** — Direct XSS candidates requiring manual testing
4. **NTLM findings** — Internal domain name disclosure, potential relay attacks
5. **Host header injection** — Password reset poisoning, cache poisoning
6. **Hidden admin panels (ffuf)** — Default creds, sensitive functionality
7. **Downloaded sensitive files** — `.env`, `.bak`, `.sql` files
8. **OAuth endpoints** — Auth flow testing, token leakage
9. **Discovered parameters** — Input validation, injection testing
10. **Virtual hosts found** — Separate application instances

---

## Web Audit Module Flag Reference

| Flag | Includes | Skip When |
|---|---|---|
| `web-basic` | httpx, robots, securitytxt, social, oauth | Never — always use |
| `web-thorough` | web-basic + paramminer, ffuf, vhost, etc. | Passive-only programs |
| `web-paramminer` | paramminer_getparams/headers/cookies | Low rate limit programs |
| `web-screenshots` | gowitness | When visual survey not needed |
| `iis-shortnames` | iis_shortnames, ffuf_shortnames | Non-IIS targets |
