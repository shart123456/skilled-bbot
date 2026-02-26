---
name: Internal Engine Modules
description: Reference for BBOT internal engine modules that run automatically and are not invoked via -m
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
# BBot Internal Engine Modules

These modules are part of BBOT's core processing pipeline. They run **automatically** during every scan — you do not invoke them via `-m`. They are listed here for reference because they appear in event graphs and scan logs.

> **Note:** You cannot enable or disable most of these via the command line. They are controlled via global config YAML (`~/.config/bbot/bbot.yml`).

---

## aggregate

**Purpose:** Deduplicates and aggregates events across the scan.

**Mechanism:** Maintains a global set of seen event hashes. When a module produces an event (e.g., `DNS_NAME: api.example.com`), `aggregate` checks if that exact event was already emitted by any module. If so, it is suppressed before being dispatched to downstream modules.

**Watched Events:** All event types
**Produced Events:** None (filter only)

**Why it matters:** Prevents the same subdomain from being re-processed by every subsequent module when multiple sources discover it. Keeps scan efficient at scale.

**Config options:**
| Option | Default | Description |
|---|---|---|
| `bbot.dedup_strategy` | `"default"` | Deduplication strategy (`default` or `none`) |

---

## dnsresolve

**Purpose:** Resolves DNS names discovered during scanning to IP addresses.

**Mechanism:** Takes every `DNS_NAME` event and performs forward DNS resolution. Produces `IP_ADDRESS` events for each resolved IP. Also performs reverse DNS (PTR) lookups on discovered IPs to find additional hostnames.

**Watched Events:** `DNS_NAME`, `IP_ADDRESS`
**Produced Events:** `IP_ADDRESS`, `DNS_NAME` (via reverse PTR lookups)

**Config options:**
| Option | Default | Description |
|---|---|---|
| `dns.threads` | `25` | DNS resolution thread count |
| `dns.timeout` | `5` | Per-query timeout in seconds |
| `dns.retries` | `1` | Number of retries per failed query |
| `dns.nameservers` | System default | Custom DNS resolvers (list of IPs) |
| `dns.brute_threads` | `1000` | Threads used by `dnsbrute` (separate) |

**Why it matters:** Bridges `DNS_NAME` → `IP_ADDRESS` in the event graph. Without `dnsresolve`, port scanning and IP-based modules receive no targets.

---

## speculate

**Purpose:** Generates "speculative" events from existing scan data without making new network requests.

**Mechanism:** Analyzes discovered events and infers additional likely targets. Examples:
- From `DNS_NAME: api.example.com` → speculates `https://api.example.com` as `URL_UNVERIFIED`
- From `OPEN_TCP_PORT: 1.2.3.4:443` → speculates `https://1.2.3.4` as `URL_UNVERIFIED`
- From an ASN range → speculates individual IPs within the range

**Watched Events:** `DNS_NAME`, `IP_ADDRESS`, `OPEN_TCP_PORT`, `IP_RANGE`, `ASN`
**Produced Events:** `URL_UNVERIFIED`, `IP_ADDRESS` (from CIDR expansion)

**Why it matters:** Drives the "chaining" behavior that makes BBOT powerful — discovered assets automatically become inputs for the next layer of modules. If `speculate` is disabled, most multi-stage scans break.

**Config options:**
| Option | Default | Description |
|---|---|---|
| `scope.report_distance` | `0` | Distance from initial target; `0` = in-scope only |

---

## excavate

**Purpose:** Extracts embedded data from HTTP responses — URLs, emails, API keys, subdomains.

**Mechanism:** When `httpx` produces an `HTTP_RESPONSE` event, `excavate` parses the response body and headers for:
- Embedded URLs (absolute and relative)
- Email addresses
- Subdomains of the target
- API endpoints
- Secret patterns (tokens, keys, credentials)
- JavaScript source map references (`//# sourceMappingURL`)

**Watched Events:** `HTTP_RESPONSE`
**Produced Events:** `URL_UNVERIFIED`, `EMAIL_ADDRESS`, `DNS_NAME`, `FINDING`

**Why it matters:** Every web page fetched by `httpx` is automatically mined for additional targets. This creates the crawling chain: `httpx` → `excavate` → new `URL_UNVERIFIED` → `httpx` (follow links).

**Config options:**
| Option | Default | Description |
|---|---|---|
| `web.spider_distance` | `0` | How many links deep to follow (0 = same-page only) |
| `web.spider_links_per_page` | `25` | Max links to follow per page |

---

## portfilter

**Purpose:** Filters open port events to suppress noise from common non-interesting ports.

**Mechanism:** After `portscan` discovers open ports, `portfilter` evaluates each `OPEN_TCP_PORT` event against a filter list. Ports that match common services already handled elsewhere (e.g., 80, 443 always handled by `httpx`) may be suppressed from the output to reduce event noise.

**Watched Events:** `OPEN_TCP_PORT`
**Produced Events:** None (filter only)

**Why it matters:** Reduces output noise. Without it, every HTTP/HTTPS port generates duplicate downstream events.

**Config options:**
| Option | Default | Description |
|---|---|---|
| `scope.in_scope_only` | `True` | Filter out-of-scope ports |

---

## unarchive

**Purpose:** Extracts files from archives discovered during scanning.

**Mechanism:** When `filedownload` saves an archive (`.zip`, `.tar.gz`, `.7z`, etc.) to disk as a `FILESYSTEM` event, `unarchive` extracts its contents and emits new `FILESYSTEM` events for each extracted file. These files are then available for analysis by `extractous` or manual inspection.

**Watched Events:** `FILESYSTEM`
**Produced Events:** `FILESYSTEM`

**Supported Archive Formats:**
- ZIP (`.zip`)
- TAR / TAR.GZ / TAR.BZ2 (`.tar`, `.tar.gz`, `.tgz`, `.tar.bz2`)
- 7-Zip (`.7z`)
- GZIP (`.gz`, standalone)

**Works with:**
```bash
bbot -t example.com -m httpx filedownload unarchive extractous
```

**Why it matters:** Archives downloaded from exposed S3 buckets or web servers often contain source code, config files, and credentials. `unarchive` feeds those contents into `extractous` automatically.

---

## Disabling Internal Modules (Global Config)

Internal modules can be partially configured via `~/.config/bbot/bbot.yml`:

```yaml
# ~/.config/bbot/bbot.yml

# DNS resolver configuration
dns:
  threads: 25
  timeout: 5
  retries: 1
  # Custom resolvers (bypass ISP DNS):
  # nameservers:
  #   - 8.8.8.8
  #   - 1.1.1.1

# Web spider depth (excavate)
web:
  spider_distance: 1       # Follow 1 link deep
  spider_links_per_page: 10

# Scope controls (speculate / portfilter)
scope:
  report_distance: 0       # Only in-scope events
  in_scope_only: true
```

---

## Event Flow Diagram

```
Initial Target (DNS_NAME / IP_ADDRESS / CIDR)
    │
    ├─> dnsresolve ──────────────────> IP_ADDRESS
    │       │
    │       └─> speculate ──────────> URL_UNVERIFIED
    │
    ├─> [scan modules] ─────────────> DNS_NAME / OPEN_TCP_PORT / HTTP_RESPONSE
    │
    ├─> portfilter ─────────────────> (filtered) OPEN_TCP_PORT
    │
    ├─> speculate ──────────────────> URL_UNVERIFIED (from OPEN_TCP_PORT)
    │
    ├─> httpx ─────────────────────> HTTP_RESPONSE
    │
    ├─> excavate ───────────────────> URL_UNVERIFIED / EMAIL_ADDRESS / FINDING
    │
    └─> aggregate ──────────────────> (deduplication at every stage)
```
