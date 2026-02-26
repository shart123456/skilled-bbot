---
name: bbot
description: |
  Execute intelligent BBOT scans for bug bounty reconnaissance that stay within program scope, respect rate limits, and use appropriate flags (passive/safe-active/thorough). Automatically configures blacklists, rate limits, and modules based on bug bounty program policies. Full reference for all 117+ BBOT modules organized by category — DNS enumeration, port scanning, web fuzzing, vulnerability scanning, cloud storage, code repositories, email/credential discovery, and output modules. Includes presets and multi-module workflows. USE WHEN user says "Run BBOT scan", "BBOT scan [COMPANY]", "start reconnaissance", "Bug Bounty and BBot", "enumerate subdomains", "scan for S3 buckets", "check subdomain takeover", "run nuclei", or requests any BBOT module by name.
---

# BBOT Bug Bounty Scanner

BBOT (Bighuge BLS OSINT Tool) v2.8.2 — recursive, modular attack surface management and OSINT framework with 117+ scanning modules.

**Installation:** `/opt/bbot/`
**Scan Output:** `/opt/bbot/scans/`
**Config:** `/opt/bbot/bbot/defaults.yml`

---

## When to Activate This Skill

- "Run BBOT scan for [COMPANY_NAME]"
- "BBOT scan [COMPANY_NAME]"
- "Start reconnaissance on [COMPANY_NAME]"
- "Enumerate subdomains for [TARGET]"
- "Check for S3 buckets / cloud assets"
- "Subdomain takeover detection"
- "Bug Bounty and BBot" + HTML program page
- Any request involving a specific BBOT module name
- "Run nuclei / dnsbrute / httpx / ffuf / etc."

---

## Core Command Syntax

```bash
# Single target
bbot -t example.com

# Multiple targets
bbot -t example.com 192.168.1.0/24 "*.example.com"

# With flags (category-based module selection)
bbot -t example.com -f subdomain-enum passive safe

# With specific modules
bbot -t example.com -m dnsbrute crt httpx nuclei

# With preset
bbot -t example.com -p subdomain-enum

# With output
bbot -t example.com -f subdomain-enum -o ~/scans/example/ -om json subdomains

# Exclude modules
bbot -t example.com -f web-thorough -em wpscan ffuf

# Exclude flags
bbot -t example.com -f web-thorough -ef aggressive deadly

# Blacklist (out-of-scope)
bbot -t example.com --blacklist admin.example.com staging.example.com

# Scope control
bbot -t "*.example.com" --strict-scope

# Dry run (preview)
bbot -t example.com -f subdomain-enum --dry-run
```

---

## Scan Type Decision Tree

```
Program Policy?
├── Passive Only    → -f subdomain-enum passive safe
├── Active Allowed  → -f subdomain-enum web-basic safe
├── Thorough OK     → -p web-thorough
└── Full Engagement → -p kitchen-sink -ef deadly
```

---

## Flags Reference (Module Categories)

| Flag | Description | Risk |
|---|---|---|
| `passive` | No direct contact with target | Zero |
| `safe` | Non-destructive active probing | Low |
| `active` | Direct target contact | Medium |
| `aggressive` | Intrusive, noisy | High |
| `deadly` | Highly intrusive (disabled by default) | Critical |
| `subdomain-enum` | Subdomain discovery | Varies |
| `web-basic` | Basic web scanning | Low |
| `web-thorough` | Thorough web testing | Medium |
| `web-paramminer` | Parameter brute-forcing | High |
| `web-screenshots` | Screenshot capture | Low |
| `cloud-enum` | Cloud asset discovery | Low |
| `code-enum` | Code repository discovery | Passive |
| `email-enum` | Email discovery | Passive |
| `portscan` | Port scanning | Active |
| `baddns` | Domain takeover checks | Active |
| `iis-shortnames` | IIS shortname detection | Active |
| `affiliates` | Check affiliate/related domains | Active |

---

## Quick Module Reference by Category

### DNS & Subdomain Enumeration
`anubisdb` `bufferoverrun` `c99` `censys_dns` `certspotter` `chaos` `crt` `crt_db` `dnsbimi` `dnsbrute` `dnsbrute_mutations` `dnscaa` `dnscommonsrv` `dnsdumpster` `dnstlsrpt` `fullhunt` `hackertarget` `leakix` `myssl` `otx` `rapiddns` `securitytrails` `shodan_dns` `sitedossier` `subdomaincenter` `subdomainradar` `trickest` `urlscan` `viewdns` `virustotal` `wayback`

### Port Scanning & Service Detection
`portscan` `sslcert` `portfilter` `ipneighbor` `fingerprintx`

### Web Scanning & Fuzzing
`httpx` `ffuf` `ffuf_shortnames` `bypass403` `iis_shortnames` `robots` `url_manipulation` `host_header` `paramminer_getparams` `paramminer_headers` `paramminer_cookies` `reflected_parameters` `smuggler` `vhost` `gowitness` `social` `oauth` `securitytxt` `newsletters` `graphql_introspection` `generic_ssrf` `code_repository`

### Vulnerability Scanning
`nuclei` `badsecrets` `baddns` `baddns_direct` `baddns_zone` `bypass403` `dotnetnuke` `ajaxpro` `telerik` `wpscan` `retirejs` `ntlm` `aspnet_bin_exposure` `medusa`

### Cloud & Storage
`bucket_amazon` `bucket_google` `bucket_microsoft` `bucket_firebase` `bucket_digitalocean` `bucket_file_enum` `azure_realm` `azure_tenant` `censys_ip` `shodan_idb`

### Code Repository Discovery
`github_org` `github_codesearch` `github_usersearch` `github_workflows` `gitlab_com` `gitlab_onprem` `postman` `postman_download` `git` `git_clone` `gitdumper` `dockerhub` `docker_pull` `google_playstore` `apkpure` `jadx`

### Email & Credentials
`emailformat` `hunterio` `skymem` `pgp` `dehashed` `credshed` `trufflehog` `passivetotal`

### IP Intelligence & Tech Detection
`ip2location` `ipstack` `digitorus` `builtwith` `ironsight` `bevigil` `wafw00f` `extractous` `filedownload`

---

## Available Presets

| Preset | Description | Use When |
|---|---|---|
| `subdomain-enum` | Passive + active subdomain discovery | Standard recon |
| `cloud-enum` | Cloud asset enumeration | Cloud-heavy targets |
| `code-enum` | Code repository discovery | Software companies |
| `email-enum` | Email address discovery | Phishing research |
| `web-basic` | Basic web scanning | Initial web recon |
| `web-thorough` | Thorough web testing | Deep web audit |
| `web-screenshots` | Screenshot capture | Visual survey |
| `tech-detect` | Technology fingerprinting | Stack identification |
| `baddns-intense` | Domain takeover intensive | Takeover hunting |
| `fast` | Quick scope-limited scan | Time-constrained |
| `kitchen-sink` | Everything | Full engagement |
| `spider` | Web spidering | Content discovery |
| `spider-intense` | Intensive spidering | Deep content crawl |

---

## Safety Controls

```bash
# Always use blacklist for out-of-scope
bbot -t "*.acme.com" \
     --blacklist admin.acme.com legacy.acme.com \
     -f subdomain-enum safe

# Rate limiting
bbot -t example.com \
     -c web_requests_per_second=10 \
     -f web-basic safe

# Strict scope (no subdomain inference)
bbot -t example.com --strict-scope

# Passive only (zero contact)
bbot -t example.com -f passive
```

---

## Supplementary Resources

- Full module reference: `read /home/shartlieba/.claude/skills/bbot/CLAUDE.md`
- DNS modules detail: `read /home/shartlieba/.claude/skills/bbot/modules/dns-subdomain-enum.md`
- Port scanning detail: `read /home/shartlieba/.claude/skills/bbot/modules/port-scanning.md`
- Web scanning detail: `read /home/shartlieba/.claude/skills/bbot/modules/web-scanning.md`
- Vuln scanning detail: `read /home/shartlieba/.claude/skills/bbot/modules/vulnerability-scanning.md`
- Cloud enumeration detail: `read /home/shartlieba/.claude/skills/bbot/modules/cloud-enumeration.md`
- Code repository detail: `read /home/shartlieba/.claude/skills/bbot/modules/code-repository.md`
- Email/credentials detail: `read /home/shartlieba/.claude/skills/bbot/modules/email-credentials.md`
- IP intelligence detail: `read /home/shartlieba/.claude/skills/bbot/modules/ip-intelligence.md`
- Output modules detail: `read /home/shartlieba/.claude/skills/bbot/modules/output-modules.md`
- Passive recon workflow: `read /home/shartlieba/.claude/skills/bbot/workflows/passive-recon.md`
- Safe active workflow: `read /home/shartlieba/.claude/skills/bbot/workflows/safe-active-recon.md`
- Subdomain takeover workflow: `read /home/shartlieba/.claude/skills/bbot/workflows/subdomain-takeover.md`
- Cloud hunt workflow: `read /home/shartlieba/.claude/skills/bbot/workflows/cloud-hunt.md`
- Vuln scan workflow: `read /home/shartlieba/.claude/skills/bbot/workflows/vuln-scan.md`
- Web app audit workflow: `read /home/shartlieba/.claude/skills/bbot/workflows/web-app-audit.md`
- Code leak hunt workflow: `read /home/shartlieba/.claude/skills/bbot/workflows/code-leak-hunt.md`
- Full engagement workflow: `read /home/shartlieba/.claude/skills/bbot/workflows/full-engagement.md`

---

**Version:** 3.0.0 (Complete Module Coverage)
**BBOT Version:** 2.8.2
**Modules Covered:** 117+ main + 23 output + 8 internal
**Last Updated:** 2026-02-25
