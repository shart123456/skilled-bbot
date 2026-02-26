---
name: DNS and Subdomain Enumeration Modules
description: Complete reference for all passive and active DNS subdomain discovery modules in BBOT
license: MIT
compatibility: MONK TAO skills system, Claude Code
metadata:
  author: shart123456
  usage: documentation file for MONK TAO skills system
  version: 0.1.0
  related files: ""
  creation date: 2026-02-26
  last modified: 2026-02-26
---
# BBot DNS & Subdomain Enumeration Modules

Complete reference for all passive and active DNS/subdomain discovery modules in BBOT.

---

## anubisdb

**Source:** `jldc.me` subdomain database (free, no API key required)

**Flags:** `passive`, `safe`, `subdomain-enum`
**Watched Events:** `DNS_NAME`
**Produced Events:** `DNS_NAME`

**Description:** Queries the jldc.me ANUBISDB subdomain database for known subdomains of the target domain. Completely passive — no contact with the target.

**No configuration options required.**

```bash
bbot -t example.com -m anubisdb
```

---

## bufferoverrun

**Source:** `dns.bufferover.run` (passive DNS database)

**Flags:** `passive`, `safe`, `subdomain-enum`
**Watched Events:** `DNS_NAME`
**Produced Events:** `DNS_NAME`

**Description:** Queries BufferOverrun's passive DNS database for historical subdomain records. No API key required.

```bash
bbot -t example.com -m bufferoverrun
```

---

## c99

**Source:** `c99.nl` API

**Flags:** `passive`, `safe`, `subdomain-enum`
**Watched Events:** `DNS_NAME`
**Produced Events:** `DNS_NAME`

**Description:** Queries the C99.nl API for known subdomains. Requires a paid API key.

**Options:**
| Option | Default | Description |
|---|---|---|
| `api_key` | `""` | C99.nl API key (required) |

**Setup:**
```bash
bbot -t example.com -m c99 -c modules.c99.api_key=YOUR_KEY
```

---

## censys_dns

**Source:** `censys.io` DNS data

**Flags:** `passive`, `safe`, `subdomain-enum`
**Watched Events:** `DNS_NAME`
**Produced Events:** `DNS_NAME`

**Description:** Queries Censys.io for DNS records associated with the target domain. Requires Censys API credentials.

**Options:**
| Option | Default | Description |
|---|---|---|
| `api_id` | `""` | Censys API ID |
| `api_secret` | `""` | Censys API Secret |

```bash
bbot -t example.com -m censys_dns \
     -c modules.censys_dns.api_id=ID \
     -c modules.censys_dns.api_secret=SECRET
```

---

## certspotter

**Source:** `sslmate.com/certspotter` certificate transparency API

**Flags:** `passive`, `safe`, `subdomain-enum`
**Watched Events:** `DNS_NAME`
**Produced Events:** `DNS_NAME`

**Description:** Queries CertSpotter's certificate transparency API for SSL certificates issued to subdomains of the target. Discovers subdomains that may not appear in DNS databases.

**Options:**
| Option | Default | Description |
|---|---|---|
| `api_key` | `""` | CertSpotter API key (free tier available) |

```bash
bbot -t example.com -m certspotter
# Or with API key for higher rate limits:
bbot -t example.com -m certspotter -c modules.certspotter.api_key=KEY
```

---

## chaos

**Source:** `chaos.projectdiscovery.io` DNS data

**Flags:** `passive`, `safe`, `subdomain-enum`
**Watched Events:** `DNS_NAME`
**Produced Events:** `DNS_NAME`

**Description:** Queries ProjectDiscovery's Chaos dataset — a regularly updated database of subdomains across the internet. Excellent coverage for popular programs.

**Options:**
| Option | Default | Description |
|---|---|---|
| `api_key` | `""` | Chaos API key (required, free from chaos.projectdiscovery.io) |

```bash
bbot -t example.com -m chaos -c modules.chaos.api_key=KEY
```

---

## crt

**Source:** `crt.sh` certificate transparency log

**Flags:** `passive`, `safe`, `subdomain-enum`
**Watched Events:** `DNS_NAME`
**Produced Events:** `DNS_NAME`

**Description:** Queries crt.sh for SSL/TLS certificates issued by public certificate authorities. Certificate transparency logs are an excellent passive subdomain source that reveals all publicly-trusted certificates including wildcards.

**No API key required. No configuration options.**

```bash
bbot -t example.com -m crt
```

**Notes:**
- Covers all CA-issued certificates since ~2013
- Often reveals wildcard and internal subdomain patterns
- May return expired certificate entries (still valuable)

---

## crt_db

**Source:** `crt.sh` PostgreSQL direct connection

**Flags:** `passive`, `safe`, `subdomain-enum`
**Watched Events:** `DNS_NAME`
**Produced Events:** `DNS_NAME`

**Description:** Alternative to `crt` — queries crt.sh directly via its PostgreSQL database for faster/more complete results. Produces the same output as `crt` but with different querying method.

```bash
bbot -t example.com -m crt_db
```

---

## dnsbimi

**Source:** Active DNS queries

**Flags:** `active`, `safe`, `subdomain-enum`
**Watched Events:** `DNS_NAME`
**Produced Events:** `DNS_NAME`

**Description:** Checks for BIMI (Brand Indicators for Message Identification) DNS records at `_bimi.[domain]`. BIMI records reveal email infrastructure and may indicate related domains.

```bash
bbot -t example.com -m dnsbimi
```

---

## dnsbrute

**Source:** Active DNS brute-force

**Flags:** `active`, `aggressive`, `subdomain-enum`
**Watched Events:** `DNS_NAME`
**Produced Events:** `DNS_NAME`

**Description:** Brute-forces subdomains by prepending wordlist entries to the target domain and resolving them using massdns. One of the most effective subdomain discovery techniques for finding unlisted subdomains.

**Dependencies:** `massdns` (system tool)

**Options:**
| Option | Default | Description |
|---|---|---|
| `wordlist` | SecLists subdomains-top1million-5000.txt (URL) | Path or URL to DNS wordlist |
| `max_depth` | `5` | Maximum recursion depth for discovered subdomains |

```bash
# Default (5000 word SecLists list)
bbot -t example.com -m dnsbrute

# Custom wordlist
bbot -t example.com -m dnsbrute \
     -c modules.dnsbrute.wordlist=/path/to/wordlist.txt

# With larger wordlist
bbot -t example.com -m dnsbrute \
     -c modules.dnsbrute.wordlist=https://raw.githubusercontent.com/danielmiessler/SecLists/master/Discovery/DNS/subdomains-top1million-110000.txt
```

**Notes:**
- massdns must be installed (`apt install massdns`)
- DNS threads configurable globally: `-c dns.brute_threads=1000`
- Recursively brutes discovered subdomains up to `max_depth`

---

## dnsbrute_mutations

**Source:** Active DNS — mutation-based

**Flags:** `active`, `aggressive`, `subdomain-enum`
**Watched Events:** `DNS_NAME`
**Produced Events:** `DNS_NAME`

**Description:** Generates mutations of discovered subdomains (e.g., `api.example.com` → `api-dev.example.com`, `api-staging.example.com`) and brute-forces those. Highly effective for finding environment-specific subdomains.

**Dependencies:** `massdns`, `alterx` (ProjectDiscovery mutation engine)

**Options:**
| Option | Default | Description |
|---|---|---|
| `wordlist` | Built-in mutation words | Path to mutation wordlist |
| `max_mutations` | `500` | Max mutations per subdomain |

```bash
bbot -t example.com -m dnsbrute dnsbrute_mutations
```

---

## dnscaa

**Source:** Active DNS

**Flags:** `active`, `safe`, `subdomain-enum`
**Watched Events:** `DNS_NAME`
**Produced Events:** `DNS_NAME`

**Description:** Queries CAA (Certification Authority Authorization) DNS records. CAA records specify which CAs can issue certificates for a domain — useful for understanding PKI infrastructure and identifying certificate pinning.

```bash
bbot -t example.com -m dnscaa
```

---

## dnscommonsrv

**Source:** Active DNS

**Flags:** `active`, `safe`, `subdomain-enum`
**Watched Events:** `DNS_NAME`
**Produced Events:** `DNS_NAME`

**Description:** Checks for common SRV (Service) DNS records like `_sip._tcp`, `_xmpp-client._tcp`, `_ldap._tcp`, `_kerberos._tcp`, etc. SRV records reveal internal service infrastructure and can expose hidden hosts.

**Common SRV Records Checked:**
- `_http._tcp` — HTTP services
- `_https._tcp` — HTTPS services
- `_ftp._tcp` — FTP
- `_sftp._tcp` — SFTP
- `_ssh._tcp` — SSH
- `_sip._tcp/_udp` — VoIP
- `_xmpp-client._tcp` — XMPP/Jabber
- `_ldap._tcp` — LDAP/AD
- `_kerberos._tcp` — Kerberos/AD
- `_caldav._tcp` — CalDAV
- `_carddav._tcp` — CardDAV

```bash
bbot -t example.com -m dnscommonsrv
```

---

## dnsdumpster

**Source:** `dnsdumpster.com`

**Flags:** `passive`, `safe`, `subdomain-enum`
**Watched Events:** `DNS_NAME`
**Produced Events:** `DNS_NAME`

**Description:** Queries DNSDumpster for subdomain and DNS record data. DNSDumpster aggregates passive DNS data and provides network mapping. No API key required.

```bash
bbot -t example.com -m dnsdumpster
```

---

## dnstlsrpt

**Source:** Active DNS

**Flags:** `active`, `safe`, `subdomain-enum`
**Watched Events:** `DNS_NAME`
**Produced Events:** `DNS_NAME`

**Description:** Checks for TLS-RPT (TLS Reporting) DNS records at `_smtp._tls.[domain]`. TLS-RPT records specify where SMTP TLS failure reports should be sent and can reveal email infrastructure.

```bash
bbot -t example.com -m dnstlsrpt
```

---

## fullhunt

**Source:** `fullhunt.io` API

**Flags:** `passive`, `safe`, `subdomain-enum`
**Watched Events:** `DNS_NAME`
**Produced Events:** `DNS_NAME`

**Description:** Queries Fullhunt.io — an attack surface management platform with extensive subdomain and asset data. Requires a Fullhunt API key (free tier available).

**Options:**
| Option | Default | Description |
|---|---|---|
| `api_key` | `""` | Fullhunt.io API key |

```bash
bbot -t example.com -m fullhunt -c modules.fullhunt.api_key=KEY
```

---

## hackertarget

**Source:** `api.hackertarget.com`

**Flags:** `passive`, `safe`, `subdomain-enum`
**Watched Events:** `DNS_NAME`
**Produced Events:** `DNS_NAME`

**Description:** Queries the HackerTarget API for subdomain and DNS data. Provides hostsearch and other useful DNS lookups. Free tier available (rate limited).

```bash
bbot -t example.com -m hackertarget
```

---

## leakix

**Source:** `leakix.net` API

**Flags:** `passive`, `safe`, `subdomain-enum`
**Watched Events:** `DNS_NAME`
**Produced Events:** `DNS_NAME`

**Description:** Queries LeakIX — a search engine that indexes exposed services and data leaks. Provides subdomain information as well as exposure data.

**Options:**
| Option | Default | Description |
|---|---|---|
| `api_key` | `""` | LeakIX API key (optional, higher limits with key) |

```bash
bbot -t example.com -m leakix
bbot -t example.com -m leakix -c modules.leakix.api_key=KEY
```

---

## myssl

**Source:** `myssl.com` certificate database

**Flags:** `passive`, `safe`, `subdomain-enum`
**Watched Events:** `DNS_NAME`
**Produced Events:** `DNS_NAME`

**Description:** Queries myssl.com for SSL certificate records. Similar to crt.sh but from a different certificate transparency aggregator, potentially providing additional coverage.

```bash
bbot -t example.com -m myssl
```

---

## otx

**Source:** `otx.alienvault.com` (AlienVault Open Threat Exchange)

**Flags:** `passive`, `safe`, `subdomain-enum`
**Watched Events:** `DNS_NAME`
**Produced Events:** `DNS_NAME`

**Description:** Queries AlienVault OTX — a community threat intelligence platform with passive DNS data and subdomain records. Excellent coverage for well-known targets.

**Options:**
| Option | Default | Description |
|---|---|---|
| `api_key` | `""` | OTX API key (free at otx.alienvault.com) |

```bash
bbot -t example.com -m otx
bbot -t example.com -m otx -c modules.otx.api_key=KEY
```

---

## rapiddns

**Source:** `rapiddns.io`

**Flags:** `passive`, `safe`, `subdomain-enum`
**Watched Events:** `DNS_NAME`
**Produced Events:** `DNS_NAME`

**Description:** Queries RapidDNS.io for subdomain records. Aggregates DNS data from multiple sources. No API key required.

```bash
bbot -t example.com -m rapiddns
```

---

## securitytrails

**Source:** `securitytrails.com` API

**Flags:** `passive`, `safe`, `subdomain-enum`
**Watched Events:** `DNS_NAME`
**Produced Events:** `DNS_NAME`

**Description:** Queries SecurityTrails — one of the most comprehensive historical DNS databases. Provides current and historical DNS records, subdomain data, and IP history. Premium API key required.

**Options:**
| Option | Default | Description |
|---|---|---|
| `api_key` | `""` | SecurityTrails API key |

```bash
bbot -t example.com -m securitytrails -c modules.securitytrails.api_key=KEY
```

---

## shodan_dns

**Source:** `shodan.io` DNS data

**Flags:** `passive`, `safe`, `subdomain-enum`
**Watched Events:** `DNS_NAME`
**Produced Events:** `DNS_NAME`

**Description:** Queries Shodan's DNS database for subdomains associated with the target domain. Shodan indexes internet-facing services and maintains DNS record history.

**Options:**
| Option | Default | Description |
|---|---|---|
| `api_key` | `""` | Shodan API key |

```bash
bbot -t example.com -m shodan_dns -c modules.shodan_dns.api_key=KEY
```

---

## sitedossier

**Source:** `sitedossier.com`

**Flags:** `passive`, `safe`, `subdomain-enum`
**Watched Events:** `DNS_NAME`
**Produced Events:** `DNS_NAME`

**Description:** Queries SiteDossier.com for subdomain data. Aggregates historical DNS and web crawl data. No API key required.

```bash
bbot -t example.com -m sitedossier
```

---

## subdomaincenter

**Source:** `api.subdomain.center`

**Flags:** `passive`, `safe`, `subdomain-enum`
**Watched Events:** `DNS_NAME`
**Produced Events:** `DNS_NAME`

**Description:** Queries the Subdomain Center API for known subdomains. A community-contributed subdomain database. No API key required.

```bash
bbot -t example.com -m subdomaincenter
```

---

## subdomainradar

**Source:** `subdomainradar.io`

**Flags:** `passive`, `safe`, `subdomain-enum`
**Watched Events:** `DNS_NAME`
**Produced Events:** `DNS_NAME`

**Description:** Queries SubdomainRadar.io for subdomain data. An additional passive subdomain discovery source.

**Options:**
| Option | Default | Description |
|---|---|---|
| `api_key` | `""` | SubdomainRadar API key (optional) |

```bash
bbot -t example.com -m subdomainradar
```

---

## trickest

**Source:** `trickest.io` subdomain database

**Flags:** `passive`, `safe`, `subdomain-enum`
**Watched Events:** `DNS_NAME`
**Produced Events:** `DNS_NAME`

**Description:** Queries Trickest's community-sourced subdomain database. Trickest collects and maintains a large dataset of internet subdomains.

**Options:**
| Option | Default | Description |
|---|---|---|
| `api_key` | `""` | Trickest API key |

```bash
bbot -t example.com -m trickest -c modules.trickest.api_key=KEY
```

---

## urlscan

**Source:** `urlscan.io` API

**Flags:** `passive`, `safe`, `subdomain-enum`
**Watched Events:** `DNS_NAME`
**Produced Events:** `DNS_NAME`, `URL_UNVERIFIED`

**Description:** Queries urlscan.io — a service that scans and analyzes websites. The database contains historical URL and DNS data from user-submitted scans and automated crawls. Returns both subdomains and URLs.

**Options:**
| Option | Default | Description |
|---|---|---|
| `api_key` | `""` | urlscan.io API key (optional, higher rate limits) |
| `urls` | `False` | Also return URL events |

```bash
bbot -t example.com -m urlscan
bbot -t example.com -m urlscan -c modules.urlscan.api_key=KEY modules.urlscan.urls=true
```

---

## viewdns

**Source:** `viewdns.info`

**Flags:** `passive`, `safe`, `subdomain-enum`
**Watched Events:** `DNS_NAME`
**Produced Events:** `DNS_NAME`

**Description:** Queries ViewDNS.info for DNS and subdomain information. Provides reverse IP lookups, DNS history, and related domain data.

```bash
bbot -t example.com -m viewdns
```

---

## virustotal

**Source:** `virustotal.com` API

**Flags:** `passive`, `safe`, `subdomain-enum`
**Watched Events:** `DNS_NAME`
**Produced Events:** `DNS_NAME`

**Description:** Queries VirusTotal's domain report API for subdomain data. VirusTotal aggregates data from passive DNS replication, URL scanning, and threat intelligence feeds.

**Options:**
| Option | Default | Description |
|---|---|---|
| `api_key` | `""` | VirusTotal API key (free tier available at virustotal.com) |

```bash
bbot -t example.com -m virustotal -c modules.virustotal.api_key=KEY
```

---

## wayback

**Source:** `web.archive.org` Wayback Machine / CDX API

**Flags:** `passive`, `safe`, `subdomain-enum`
**Watched Events:** `DNS_NAME`
**Produced Events:** `DNS_NAME`, `URL_UNVERIFIED`

**Description:** Queries the Wayback Machine's CDX (Capture Index) API for historical URLs and subdomains. Excellent for discovering:
- Subdomains that existed in the past
- Old API endpoints
- Hidden paths that may still be active
- Decommissioned services

**Options:**
| Option | Default | Description |
|---|---|---|
| `urls` | `False` | Also emit URL events from wayback records |

```bash
bbot -t example.com -m wayback
bbot -t example.com -m wayback -c modules.wayback.urls=true
```

---

## Combined DNS Enumeration Commands

### Maximum Passive Coverage (No API Keys)
```bash
bbot -t example.com \
     -m anubisdb bufferoverrun crt crt_db dnsdumpster hackertarget \
        myssl otx rapiddns sitedossier subdomaincenter urlscan viewdns wayback \
     -n passive_dns_enum
```

### Full Passive Coverage (With API Keys)
```bash
bbot -t example.com \
     -f subdomain-enum passive safe \
     -c modules.securitytrails.api_key=KEY \
        modules.virustotal.api_key=KEY \
        modules.chaos.api_key=KEY \
        modules.certspotter.api_key=KEY \
     -n full_passive_enum
```

### Active + Passive Subdomain Enumeration
```bash
bbot -t example.com \
     -f subdomain-enum \
     -c dns.brute_threads=1000 \
     -n full_subdomain_enum
```

### Aggressive Brute-Force Only
```bash
bbot -t example.com \
     -m dnsbrute dnsbrute_mutations \
     -c modules.dnsbrute.wordlist=/opt/wordlists/dns/subdomains-top1million-110000.txt \
     -n aggressive_brute
```
