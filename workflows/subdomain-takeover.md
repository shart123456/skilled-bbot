# Workflow: Subdomain Takeover Detection

**Risk Level:** Low — checking CNAME targets, no exploitation
**Use When:** Hunting subdomain takeovers as a bug class, high-value for bug bounty

---

## Overview

Subdomain takeover occurs when a CNAME record points to an external service that has been deleted or unclaimed. An attacker can register the service and take ownership of the subdomain, serving malicious content under the target's domain.

**Modules Used:** baddns, baddns_direct, baddns_zone, crt, dnsdumpster, anubisdb, dnsbrute, httpx

---

## Step 1: Maximum Subdomain Discovery

The more subdomains found, the more takeover opportunities:

```bash
COMPANY="acme"
TARGET="acme.com"

bbot -t $TARGET \
     -f subdomain-enum \
     -c dns.brute_threads=1000 \
        modules.dnsbrute.max_depth=5 \
     -om json subdomains \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_subdomain_discovery
```

---

## Step 2: Takeover Detection (All baddns Variants)

```bash
# Run all three baddns modules for comprehensive coverage
bbot -t $TARGET \
     -f subdomain-enum baddns \
     -c modules.baddns.only_high_confidence=false \
     -om json web_report \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_takeover_check
```

---

## Step 3: Intensive Takeover Scan (Preset)

```bash
# Use the baddns-intense preset for maximum coverage
bbot -t $TARGET -p baddns-intense \
     -om json subdomains web_report \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_baddns_intense
```

---

## Step 4: Check Unresolved DNS Names

Unresolved CNAMEs are primary takeover targets:

```bash
# Focus on DNS_NAME_UNRESOLVED events
bbot -t $TARGET \
     -m anubisdb crt dnsdumpster dnsbrute baddns baddns_direct baddns_zone \
     -om json \
     -o /tmp/takeover_check/

# Extract unresolved names
cat /tmp/takeover_check/*/output.ndjson | \
  jq -r 'select(.type=="DNS_NAME_UNRESOLVED") | .data'
```

---

## Understanding baddns Results

### FINDING Event Levels
- **HIGH** — Very likely takeable (CNAME → unclaimed service, confirmed open registration)
- **MEDIUM** — Possibly takeable (CNAME → service with unclear status)
- **LOW** — Worth investigating (indirect indicators)

### Services Detected (50+ providers)

| Service | CNAME Pattern | Claim Method |
|---|---|---|
| GitHub Pages | `*.github.io` | Create GitHub Page for repo |
| Heroku | `*.herokudns.com` | Create Heroku app with same name |
| Azure App Service | `*.azurewebsites.net` | Create Azure web app |
| Netlify | `*.netlify.app` | Create Netlify site |
| Vercel | `*.vercel.app` | Create Vercel project |
| AWS S3 | `*.s3.amazonaws.com` | Create S3 bucket |
| AWS CloudFront | `*.cloudfront.net` | Create CloudFront distribution |
| Fastly | `*.fastly.net` | Create Fastly service |
| Ghost | `*.ghost.io` | Create Ghost.io blog |
| Shopify | `*.myshopify.com` | Create Shopify store |
| Tumblr | `*.tumblr.com` | Create Tumblr blog |
| Zendesk | `*.zendesk.com` | Create Zendesk account |

---

## Zone Takeover (DNS Level)

`baddns_zone` checks for nameserver-level takeovers — more impactful than subdomain takeovers:

```bash
# Zone takeover only
bbot -t $TARGET \
     -m baddns_zone \
     -om json \
     -o /tmp/zone_takeover/

# Signs of zone takeover vulnerability:
# NS records pointing to unregistered nameservers
# e.g., ns1.acme.com NXDOMAIN (namespace not registered)
```

---

## Manual CNAME Investigation

After BBOT, manually investigate specific CNAME chains:

```bash
# Check CNAME chain for a subdomain
dig CNAME old-service.acme.com +short

# Check if the endpoint is registered
curl -sI https://service-name.herokudns.com/ 2>&1 | head -5

# Use dnsx for bulk CNAME resolution
cat subdomains.txt | dnsx -cname -o cnames.txt

# Find unresolved CNAMEs
cat cnames.txt | grep -v "^#" | while IFS= read -r line; do
  CNAME=$(echo "$line" | awk '{print $2}')
  if ! host "$CNAME" > /dev/null 2>&1; then
    echo "UNRESOLVED CNAME: $line"
  fi
done
```

---

## Full Takeover Hunt (Combined)

```bash
COMPANY="acme"
TARGET="acme.com"

bbot -t $TARGET \
     -f subdomain-enum baddns passive \
     -c dns.brute_threads=1000 \
        modules.baddns.only_high_confidence=false \
        modules.dnsbrute.max_depth=5 \
        modules.dnsbrute.wordlist=/opt/wordlists/dns/best-dns-wordlist.txt \
     -om json subdomains web_report \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_full_takeover_hunt
```

---

## Analyzing Takeover Results

```bash
LATEST=$(ls -td ~/bug_bounty/$COMPANY/bbot_scans/*/ | head -1)

# Extract all takeover findings
jq -r 'select(.type=="VULNERABILITY" or .type=="FINDING") |
  select(.module | startswith("baddns")) |
  "\(.data.host // .data) → \(.data.description // "")"' \
  $LATEST/output.ndjson

# High confidence only
jq -r 'select(.type=="VULNERABILITY") |
  select(.data.severity=="HIGH" or .data.severity=="CRITICAL") |
  .data.host' \
  $LATEST/output.ndjson

# All CNAME targets (to investigate manually)
jq -r 'select(.type=="DNS_NAME") |
  select(.tags | contains(["cname"])) |
  .data' \
  $LATEST/output.ndjson
```

---

## POC Template After Finding Takeover

```
Title: Subdomain Takeover on [subdomain].acme.com

Description:
The subdomain [subdomain].acme.com has a CNAME record pointing to
[service-endpoint] which is currently unclaimed and available for registration.

Steps to Reproduce:
1. Verify CNAME: dig CNAME [subdomain].acme.com
2. Result shows: [subdomain].acme.com CNAME [service-endpoint]
3. [service-endpoint] returns NXDOMAIN or service-not-found error
4. An attacker can register [service] and serve arbitrary content

Impact:
- Serve phishing pages under trusted [acme.com] domain
- Steal session cookies via malicious JS
- Bypass CSP/CORS policies
- Full XSS under target origin

Severity: HIGH
CVSS: ~8.8
```
