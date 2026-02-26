# BBOT Complete Reference — CLAUDE.md

BBOT (Bighuge BLS OSINT Tool) v2.8.2 — complete technical reference for all modules, configuration options, event types, and advanced usage patterns.

---

## Architecture Overview

BBOT operates on an **event-driven pipeline**:

```
Target Input
    └─> Internal Modules (DNS resolution, cloud check, excavate, speculate)
            └─> Main Modules (watch for specific event types)
                    └─> Output Modules (receive all events)
```

### Internal Modules (Always Active)
| Module | Function |
|---|---|
| `dnsresolve` | Resolves all DNS_NAME events to IPs |
| `cloudcheck` | Tags events with cloud provider information |
| `excavate` | Extracts URLs, emails, subdomains from HTTP response bodies |
| `speculate` | Infers new events from existing ones (e.g., OPEN_TCP_PORT → URL) |
| `aggregate` | Summarizes scan results at completion |
| `unarchive` | Extracts and processes compressed files |
| `base` | Base class all modules inherit from |

---

## Complete Event Type Reference

| Event Type | Description | Produced By |
|---|---|---|
| `DNS_NAME` | Fully-qualified domain name | crt, dnsbrute, anubisdb, etc. |
| `DNS_NAME_UNRESOLVED` | CNAME target that doesn't resolve | dnsresolve |
| `IP_ADDRESS` | IPv4 or IPv6 address | dnsresolve, portscan |
| `IP_RANGE` | CIDR range | speculate, ipneighbor |
| `OPEN_TCP_PORT` | Host:port with open TCP | portscan |
| `URL` | Verified HTTP/HTTPS URL | httpx, speculate |
| `URL_UNVERIFIED` | Unverified URL (extracted from content) | excavate, robots, wayback |
| `HTTP_RESPONSE` | Full HTTP response data | httpx |
| `EMAIL_ADDRESS` | Email address | hunterio, sslcert, pgp |
| `STORAGE_BUCKET` | Cloud storage bucket | bucket_amazon, bucket_google, etc. |
| `CODE_REPOSITORY` | Source code repository URL | github_org, postman, git |
| `TECHNOLOGY` | Detected technology/software | ironsight, nuclei, httpx |
| `FINDING` | Notable finding (non-vuln) | baddns, badsecrets, ntlm, etc. |
| `VULNERABILITY` | Confirmed vulnerability | nuclei, badsecrets, wpscan, etc. |
| `PROTOCOL` | Service running on port | fingerprintx |
| `GEOLOCATION` | IP geolocation data | ip2location, ipstack |
| `WAF` | Web Application Firewall | wafw00f |
| `SOCIAL` | Social media profile URL | social |
| `ASN` | Autonomous System Number | cloudcheck |
| `ORG_STUB` | Organization identifier | speculate |
| `VHOST` | Virtual hostname | vhost |
| `WEB_PARAMETER` | Web parameter name | paramminer_* |
| `WEBSCREENSHOT` | Screenshot data (base64) | gowitness |
| `FILESYSTEM` | Local file path | filedownload |
| `USERNAME` | Username/account | github_usersearch |
| `PASSWORD` | Credential | dehashed |
| `RAW_DNS_RECORD` | Raw DNS record data | dnsresolve |
| `MOBILE_APP` | Mobile application | google_playstore |

---

## Global Configuration Reference

Located at `/opt/bbot/bbot/defaults.yml`

### Core Settings
```yaml
# BBOT home directory
home: ~/.bbot

# Keep N scan results (auto-cleanup old scans)
keep_scans: 20

# Status update interval
status_frequency: 15

# Include raw file/folder data in events
file_blobs: false
folder_blobs: false
```

### Scope Settings
```yaml
scope:
  # Strict = subdomains not in scope unless explicitly listed
  strict: false
  # Distance from targets to include in output (0 = in-scope only)
  report_distance: 0
  # Distance to search from targets
  search_distance: 0
```

### DNS Settings
```yaml
dns:
  # Disable DNS resolution entirely
  disable: false
  # Minimal = only A/AAAA records, no CNAME following
  minimal: false
  # Concurrent DNS resolution threads
  threads: 25
  # DNS brute-force threads (for dnsbrute module)
  brute_threads: 1000
  # DNS resolution timeout
  timeout: 5
  retries: 1
  # Number of wildcard tests per domain
  wildcard_tests: 10
  # Abort if this many consecutive NXDOMAIN responses
  abort_threshold: 50
  # How far from targets to follow DNS
  search_distance: 1
  # Max depth of DNS chain to follow
  runaway_limit: 5
```

### Web Settings
```yaml
web:
  # HTTP proxy
  http_proxy: ""
  # Browser user agent
  user_agent: "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36..."
  # Web spider depth (0 = disabled)
  spider_distance: 0
  spider_depth: 1
  spider_links_per_page: 25
  # HTTP timeouts
  http_timeout: 10
  httpx_timeout: 5
  # Custom headers for all requests
  http_headers: {}
  # API and HTTP retry settings
  api_retries: 2
  http_retries: 1
  # Rate limit retry delay on 429 responses
  429_sleep_interval: 30
  # SSL verification
  ssl_verify: false
  # Global rate limit
  web_requests_per_second: 0  # 0 = unlimited
```

### Module Engine Settings
```yaml
# Enable/disable core internal modules
speculate: true
excavate: true
aggregate: true
dnsresolve: true
cloudcheck: true

# URL processing
url_querystring_remove: true   # Remove query strings from URLs
url_querystring_collapse: true  # Collapse duplicate URLs with different params

# Module timeouts
module_handle_event_timeout: 3600    # 60 min per event
module_handle_batch_timeout: 7200    # 2 hours per batch
```

---

## Command Line Reference

```bash
# Show all modules
bbot -l

# Show all flags
bbot --list-flags

# Show specific module info
bbot -m nuclei --help

# Show preset contents
bbot -p subdomain-enum --dry-run

# Verbose output
bbot -t example.com -f subdomain-enum -v

# Debug mode
bbot -t example.com -f subdomain-enum -d

# Force rescan (don't use cached results)
bbot -t example.com -f subdomain-enum --force

# Resume interrupted scan
bbot --resume example_scan_id

# Custom config file
bbot -t example.com -f subdomain-enum --config /path/to/custom.yml
```

---

## Advanced Configuration via YAML

```bash
# Create custom preset
cat > /opt/my_bug_bounty_preset.yml << 'EOF'
description: Custom bug bounty preset
modules:
  - crt
  - anubisdb
  - dnsdumpster
  - dnsbrute
  - httpx
  - sslcert
  - robots
  - securitytxt
  - badsecrets
  - baddns
  - nuclei

config:
  modules:
    dnsbrute:
      wordlist: /opt/wordlists/dns/best-dns-wordlist.txt
    nuclei:
      mode: technology
      severity: critical,high
      ratelimit: 50
    httpx:
      threads: 50
      store_responses: true

  dns:
    brute_threads: 1000

  web:
    http_timeout: 15
    web_requests_per_second: 20
EOF

# Use custom preset
bbot -t example.com -p /opt/my_bug_bounty_preset.yml
```

---

## Output Processing Cookbook

### Get All Subdomains (Deduplicated)
```bash
cat output.ndjson | jq -r 'select(.type=="DNS_NAME") | .data' | sort -u

# With scope distance filter (in-scope only)
cat output.ndjson | jq -r 'select(.type=="DNS_NAME") |
  select(.scope_distance==0) | .data' | sort -u
```

### Get All URLs
```bash
cat output.ndjson | jq -r 'select(.type=="URL") | .data' | sort -u

# HTTPS only
cat output.ndjson | jq -r 'select(.type=="URL") |
  select(.data | startswith("https://")) | .data' | sort -u
```

### Get Vulnerabilities by Severity
```bash
# Critical and High only
cat output.ndjson | jq 'select(.type=="VULNERABILITY") |
  select(.data.severity=="CRITICAL" or .data.severity=="HIGH")'

# All by severity
cat output.ndjson | jq -r 'select(.type=="VULNERABILITY") |
  "\(.data.severity // "UNKNOWN"): \(.data.name // .module) - \(.data.host // "")"' | \
  sort | uniq
```

### Technology Detection Summary
```bash
cat output.ndjson | jq -r 'select(.type=="TECHNOLOGY") | .data' | sort -u

# With host
cat output.ndjson | jq -r 'select(.type=="TECHNOLOGY") |
  "\(.data): \(.host // "")"' | sort -u
```

### Open Port Summary
```bash
# By port number
cat output.ndjson | jq -r 'select(.type=="OPEN_TCP_PORT") | .data' | \
  awk -F: '{print $NF}' | sort -n | uniq -c | sort -rn

# Full host:port list
cat output.ndjson | jq -r 'select(.type=="OPEN_TCP_PORT") | .data' | sort -u
```

### Event Statistics
```bash
cat output.ndjson | jq -r '.type' | sort | uniq -c | sort -rn
```

---

## Module Interaction Map

```
DNS_NAME
├─> anubisdb, bufferoverrun, crt, dnsdumpster, etc. → DNS_NAME
├─> dnsbrute, dnsbrute_mutations → DNS_NAME
├─> github_codesearch → CODE_REPOSITORY
├─> hunterio → EMAIL_ADDRESS, DNS_NAME
├─> bucket_amazon/google/microsoft → STORAGE_BUCKET
├─> azure_realm, azure_tenant → FINDING, DNS_NAME
├─> shodan_dns → DNS_NAME
└─> wayback, urlscan → DNS_NAME, URL_UNVERIFIED

IP_ADDRESS
├─> portscan → OPEN_TCP_PORT
├─> ipneighbor → IP_ADDRESS
├─> ip2location, ipstack → GEOLOCATION
├─> shodan_idb → OPEN_TCP_PORT, VULNERABILITY
└─> censys_ip → OPEN_TCP_PORT, PROTOCOL

OPEN_TCP_PORT
├─> sslcert → DNS_NAME, EMAIL_ADDRESS
├─> httpx → URL, HTTP_RESPONSE
└─> fingerprintx → PROTOCOL

URL / HTTP_RESPONSE
├─> nuclei → FINDING, VULNERABILITY, TECHNOLOGY
├─> badsecrets → FINDING, VULNERABILITY, TECHNOLOGY
├─> ffuf → URL_UNVERIFIED
├─> robots, securitytxt → URL_UNVERIFIED, EMAIL_ADDRESS
├─> social → SOCIAL
├─> wafw00f → WAF
├─> wpscan → FINDING, VULNERABILITY
├─> retirejs → VULNERABILITY
├─> paramminer_* → WEB_PARAMETER
└─> graphql_introspection → FINDING

CODE_REPOSITORY
├─> git_clone → (disk)
├─> trufflehog → FINDING
├─> github_workflows → FINDING
└─> postman_download → FINDING

STORAGE_BUCKET
└─> bucket_file_enum → FINDING, URL_UNVERIFIED
```

---

## Troubleshooting

### "massdns not found"
```bash
apt install massdns
# Or build from source:
git clone https://github.com/blechschmidt/massdns
cd massdns && make
sudo cp bin/massdns /usr/local/bin/
```

### "nuclei not found"
```bash
# Auto-installs on first run, or manually:
go install -v github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest
```

### Rate limit errors
```bash
# Add global rate limit
bbot -t example.com -f web-basic \
     -c web_requests_per_second=5
```

### DNS timeout issues
```bash
# Reduce threads and increase timeout
bbot -t example.com -f subdomain-enum \
     -c dns.threads=10 \
        dns.timeout=10 \
        dns.retries=2
```

### Scan not finding expected results
```bash
# Check wildcard detection isn't filtering legit results
bbot -t example.com -f subdomain-enum \
     -c dns.wildcard_tests=0

# Force strict scope off
bbot -t example.com -f subdomain-enum \
     -c scope.strict=false \
        scope.search_distance=2
```

### Resume interrupted scan
```bash
# Find scan ID
ls ~/.bbot/scans/

# Resume
bbot --resume scan_id_here
```

---

## Key File Paths

| Path | Description |
|---|---|
| `/opt/bbot/bbot/modules/` | All main module files |
| `/opt/bbot/bbot/modules/output/` | Output module files |
| `/opt/bbot/bbot/modules/internal/` | Internal module files |
| `/opt/bbot/bbot/presets/` | Built-in preset YAML files |
| `/opt/bbot/bbot/defaults.yml` | Default configuration |
| `/opt/bbot/bbot/wordlists/` | Built-in wordlists |
| `~/.bbot/scans/` | Scan output directory |
| `~/.bbot/cache/` | Module download cache |
| `~/.config/bbot/bbot.yml` | User configuration file |

---

## Module Files Index

All module source files at `/opt/bbot/bbot/modules/`:

```
ajaxpro.py          anubisdb.py         apkpure.py
aspnet_bin_exposure.py  azure_realm.py  azure_tenant.py
baddns_direct.py    baddns.py           baddns_zone.py
badsecrets.py       bevigil.py          bucket_amazon.py
bucket_digitalocean.py  bucket_file_enum.py  bucket_firebase.py
bucket_google.py    bucket_microsoft.py  bufferoverrun.py
builtwith.py        bypass403.py        c99.py
censys_dns.py       censys_ip.py        certspotter.py
chaos.py            code_repository.py  credshed.py
crt_db.py           crt.py              dehashed.py
digitorus.py        dnsbimi.py          dnsbrute_mutations.py
dnsbrute.py         dnscaa.py           dnscommonsrv.py
dnsdumpster.py      dnstlsrpt.py        dockerhub.py
docker_pull.py      dotnetnuke.py       emailformat.py
extractous.py       ffuf.py             ffuf_shortnames.py
filedownload.py     fingerprintx.py     fullhunt.py
generic_ssrf.py     git_clone.py        gitdumper.py
github_codesearch.py  github_org.py     github_usersearch.py
github_workflows.py  gitlab_com.py      gitlab_onprem.py
git.py              google_playstore.py  gowitness.py
graphql_introspection.py  hackertarget.py  host_header.py
httpx.py            hunterio.py         hunt.py
iis_shortnames.py   ip2location.py      ipneighbor.py
ipstack.py          ironsight.py        jadx.py
leakix.py           medusa.py           myssl.py
newsletters.py      ntlm.py             nuclei.py
oauth.py            otx.py              paramminer_cookies.py
paramminer_getparams.py  paramminer_headers.py  passivetotal.py
pgp.py              portfilter.py       portscan.py
postman_download.py  postman.py          rapiddns.py
reflected_parameters.py  retirejs.py    robots.py
securitytrails.py   securitytxt.py      shodan_dns.py
shodan_idb.py       sitedossier.py      skymem.py
smuggler.py         social.py           sslcert.py
subdomaincenter.py  subdomainradar.py   telerik.py
trickest.py         trufflehog.py       url_manipulation.py
urlscan.py          vhost.py            viewdns.py
virustotal.py       wafw00f.py          wayback.py
wpscan.py
```
