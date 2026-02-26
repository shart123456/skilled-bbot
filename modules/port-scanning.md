# BBot Port Scanning & Service Detection Modules

Complete reference for port scanning, service fingerprinting, and SSL certificate modules.

---

## portscan

**Description:** Port scan with masscan. Scans for open TCP ports across discovered IP addresses, IP ranges, and DNS names.

**Flags:** `active`, `portscan`, `safe`
**Watched Events:** `IP_ADDRESS`, `IP_RANGE`, `DNS_NAME`
**Produced Events:** `OPEN_TCP_PORT`

**Dependencies:** `masscan` (system tool)

**Options:**
| Option | Default | Description |
|---|---|---|
| `top_ports` | `100` | Scan the top N most common ports (uses masscan's port list) |
| `ports` | `""` | Custom port list (e.g., `"80,443,8080-8090"`) — overrides `top_ports` |
| `rate` | `300` | Masscan packets-per-second rate |
| `wait` | `5` | Seconds to wait after scan for late packets |
| `ping_first` | `False` | Ping targets before scanning (skip unresponsive hosts) |
| `ping_only` | `False` | Only ping, don't port scan (host discovery only) |
| `adapter` | `""` | Network interface to use (e.g., `eth0`) |
| `adapter_ip` | `""` | Source IP for scanning |
| `adapter_mac` | `""` | Source MAC address |
| `router_mac` | `""` | Router/gateway MAC address |
| `module_timeout` | `259200` | Maximum scan duration in seconds (3 days) |

**Usage Examples:**
```bash
# Default (top 100 ports)
bbot -t example.com -m portscan

# Top 1000 ports
bbot -t example.com -m portscan -c modules.portscan.top_ports=1000

# Specific ports
bbot -t example.com -m portscan -c modules.portscan.ports="22,80,443,3389,8080,8443"

# Full port range (all 65535)
bbot -t example.com -m portscan -c modules.portscan.ports="1-65535"

# Fast rate
bbot -t example.com -m portscan -c modules.portscan.rate=1000

# With ping discovery first
bbot -t example.com -m portscan -c modules.portscan.ping_first=true

# Specific network interface
bbot -t example.com -m portscan \
     -c modules.portscan.adapter=eth0 \
        modules.portscan.rate=500
```

**Port Discovery Chaining:**
portscan integrates with the BBOT event pipeline — discovered `OPEN_TCP_PORT` events automatically trigger:
- `sslcert` (if configured) — retrieves SSL certs from HTTPS ports
- `httpx` — HTTP probes open web ports
- `fingerprintx` — service fingerprinting

**Notes:**
- masscan requires root or CAP_NET_RAW capability
- Default rate (300 pps) is conservative — increase for faster scans on permitted targets
- `module_timeout: 259200` (3 days) allows large subnet scans
- Combined with `ipneighbor` for expanded IP coverage

---

## sslcert

**Description:** Connects to open TCP ports and retrieves SSL/TLS certificates. Extracts Subject Alternative Names (SANs) and email addresses from certificates, which often reveal additional subdomains and infrastructure.

**Flags:** `affiliates`, `subdomain-enum`, `email-enum`, `active`, `safe`, `web-basic`
**Watched Events:** `OPEN_TCP_PORT`
**Produced Events:** `DNS_NAME`, `EMAIL_ADDRESS`

**Dependencies:** `openssl` (system), `pyOpenSSL~=25.3.0` (Python)

**Options:**
| Option | Default | Description |
|---|---|---|
| `timeout` | `5.0` | SSL connection timeout in seconds |
| `skip_non_ssl` | `True` | Skip ports that don't respond to SSL handshake |

**Usage Examples:**
```bash
# Run after portscan
bbot -t example.com -m portscan sslcert

# With increased timeout for slow hosts
bbot -t example.com -m portscan sslcert \
     -c modules.sslcert.timeout=10

# With portscan for all discovered ports
bbot -t example.com -f portscan web-basic -m sslcert
```

**Why This Module Matters:**
SSL certificates often contain:
- Subject Alternative Names (SANs) listing all valid hostnames — reveals internal hostnames
- Wildcard entries (e.g., `*.internal.example.com`)
- Email addresses of certificate owners
- Organizational unit information (OU) that maps infrastructure
- Certificates shared across multiple domains (pivoting)

**Data Flow:**
```
OPEN_TCP_PORT:443 → sslcert → DNS_NAME (SANs from certificate)
                             → EMAIL_ADDRESS (cert contact email)
```

---

## portfilter

**Description:** Filters out open ports on cloud provider or CDN IP ranges from the event stream. Prevents BBOT from treating CDN/cloud IPs as interesting targets, reducing noise.

**Flags:** `active`, `safe`
**Watched Events:** `OPEN_TCP_PORT`
**Produced Events:** (filters/suppresses events)

**Options:**
| Option | Default | Description |
|---|---|---|
| `filter_asns` | `[AWS, Azure, GCP, Cloudflare, Fastly, Akamai ASNs]` | ASNs to filter ports from |

**Usage:**
```bash
# Used internally when cloud/CDN IPs detected
bbot -t example.com -f subdomain-enum web-basic -m portfilter
```

**Notes:**
- Works with `cloudcheck` internal module to identify cloud-owned IPs
- Prevents masscan from scanning CDN IPs that don't belong to the target
- Helps avoid testing infrastructure shared with other tenants

---

## ipneighbor

**Description:** Looks for additional IP addresses in subnets surrounding discovered IPs. Useful for finding adjacent infrastructure that may belong to the same organization.

**Flags:** `active`, `aggressive`, `subdomain-enum`
**Watched Events:** `IP_ADDRESS`
**Produced Events:** `IP_ADDRESS`

**Options:**
| Option | Default | Description |
|---|---|---|
| `num_bits` | `8` | Number of bits to expand the subnet (8 = /24 range) |

**Usage Examples:**
```bash
# Find neighbors of discovered IPs
bbot -t example.com -m ipneighbor

# Expand wider (16 bits = /16 range — use carefully)
bbot -t example.com -m ipneighbor -c modules.ipneighbor.num_bits=16
```

**Warning:** Using large `num_bits` values dramatically expands scope. Always check with `--dry-run` and ensure out-of-scope IPs are blacklisted.

---

## fingerprintx

**Description:** Service fingerprinting for open TCP ports. Identifies the service running on each port (SSH version, FTP banner, SMTP greeting, etc.) and produces detailed service information.

**Flags:** `active`, `safe`
**Watched Events:** `OPEN_TCP_PORT`
**Produced Events:** `PROTOCOL`

**Dependencies:** `fingerprintx` binary (downloaded from GitHub releases)

**Options:**
| Option | Default | Description |
|---|---|---|
| `version` | (latest) | Fingerprintx binary version |

**Supported Service Fingerprinting:**
- SSH (version detection, host key algorithm)
- FTP (banner grabbing)
- SMTP/IMAP/POP3 (banner, STARTTLS support)
- RDP (encryption type, protocol version)
- SMB (version, negotiated dialect)
- LDAP/Active Directory
- MySQL/PostgreSQL/MSSQL
- Redis/Memcached
- Elasticsearch
- Kafka/Zookeeper
- VNC
- And 30+ more protocols

**Usage Examples:**
```bash
# Fingerprint all open ports
bbot -t example.com -m portscan fingerprintx

# Full infrastructure discovery
bbot -t example.com \
     -m portscan sslcert fingerprintx httpx \
     -c modules.portscan.top_ports=1000
```

---

## Complete Port Scanning Pipeline

### Standard Port Scan + Web Discovery
```bash
bbot -t example.com \
     -m portscan sslcert httpx fingerprintx \
     -c modules.portscan.top_ports=1000 \
        modules.portscan.rate=500 \
     -n port_web_discovery
```

### Full Port Range + All Services
```bash
bbot -t 192.168.1.0/24 \
     -m portscan sslcert fingerprintx httpx \
     -c modules.portscan.ports="1-65535" \
        modules.portscan.rate=1000 \
        modules.portscan.ping_first=true \
     -n full_port_scan
```

### Stealth Port Discovery
```bash
# Ping sweep first, then selective port scan
bbot -t example.com \
     -m portscan \
     -c modules.portscan.ping_first=true \
        modules.portscan.ports="80,443,22,21,25,53,110,143,3306,3389,5432,6379,8080,8443,9200" \
        modules.portscan.rate=100 \
     -n stealth_scan
```

### Event Flow Visualization
```
DNS_NAME/IP_RANGE
    └─> portscan (masscan)
            └─> OPEN_TCP_PORT
                    ├─> sslcert → DNS_NAME (SAN entries), EMAIL_ADDRESS
                    ├─> httpx   → URL, HTTP_RESPONSE
                    ├─> fingerprintx → PROTOCOL (service details)
                    └─> portfilter (suppresses CDN ports)
```
