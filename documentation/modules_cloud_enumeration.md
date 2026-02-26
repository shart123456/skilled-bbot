---
name: Cloud Enumeration Modules
description: Complete reference for cloud provider asset discovery, storage bucket enumeration, and cloud infrastructure modules
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
# BBot Cloud & Storage Enumeration Modules

Complete reference for cloud provider asset discovery, storage bucket enumeration, and cloud infrastructure modules.

---

## bucket_amazon

**Description:** Discovers and tests Amazon S3 buckets related to the target domain. Generates bucket name permutations based on the target domain, checks if they exist, and identifies publicly accessible buckets.

**Flags:** `active`, `safe`, `cloud-enum`, `web-basic`
**Watched Events:** `DNS_NAME`, `STORAGE_BUCKET`
**Produced Events:** `STORAGE_BUCKET`, `FINDING`

**Options:**
| Option | Default | Description |
|---|---|---|
| `permutations` | `False` | Generate bucket name permutations (significantly expands coverage) |

**Bucket Name Patterns Tested:**
- `[domain]` → `example.com`
- `[name]` → `example`
- `[name]-[env]` → `example-prod`, `example-dev`, `example-staging`
- `[name]-[service]` → `example-backup`, `example-assets`, `example-logs`
- `[env]-[name]` → `prod-example`, `dev-example`
- `[name].[tld]` → `example.com` (bucket name with dot)

**S3 Checks Performed:**
- Bucket existence (HTTP HEAD/GET)
- Public list access (ACL misconfiguration)
- Public read access
- Public write access

**Usage Examples:**
```bash
# Basic S3 discovery
bbot -t example.com -m bucket_amazon

# With permutations (more thorough)
bbot -t example.com -m bucket_amazon \
     -c modules.bucket_amazon.permutations=true

# With file download for open buckets
bbot -t example.com -m bucket_amazon bucket_file_enum
```

**Regions Covered:** All AWS regions are tested (us-east-1, us-west-2, eu-west-1, ap-southeast-1, etc.)

---

## bucket_google

**Description:** Discovers and tests Google Cloud Storage (GCS) buckets related to the target. Similar to `bucket_amazon` but targets `storage.googleapis.com`.

**Flags:** `active`, `safe`, `cloud-enum`, `web-basic`
**Watched Events:** `DNS_NAME`, `STORAGE_BUCKET`
**Produced Events:** `STORAGE_BUCKET`, `FINDING`

**Options:**
| Option | Default | Description |
|---|---|---|
| `permutations` | `False` | Generate GCS bucket name permutations |

**GCS Checks:**
- Bucket existence
- Public list access (`AllUsers` ACL)
- Public read/write misconfiguration

**Usage Examples:**
```bash
bbot -t example.com -m bucket_google

# With permutations
bbot -t example.com -m bucket_google \
     -c modules.bucket_google.permutations=true
```

---

## bucket_microsoft

**Description:** Discovers and tests Microsoft Azure Blob Storage containers related to the target. Targets `blob.core.windows.net` storage accounts.

**Flags:** `active`, `safe`, `cloud-enum`, `web-basic`
**Watched Events:** `DNS_NAME`, `STORAGE_BUCKET`
**Produced Events:** `STORAGE_BUCKET`, `FINDING`

**Options:**
| Option | Default | Description |
|---|---|---|
| `permutations` | `False` | Generate Azure storage account name permutations |

**Azure Checks:**
- Storage account existence
- Public blob container access
- Anonymous read access
- Full public access (list + read)

**Azure Storage URL Pattern:**
`https://[storageaccount].blob.core.windows.net/[container]/`

**Usage Examples:**
```bash
bbot -t example.com -m bucket_microsoft

# With permutations
bbot -t example.com -m bucket_microsoft \
     -c modules.bucket_microsoft.permutations=true
```

---

## Azure Blob Storage Deep Dive (bucket_microsoft)

> **Module alias note:** In BBOT, `bucket_microsoft` is the canonical module name for Azure Blob Storage enumeration. The alias `bucket_azure` also works in some versions but `bucket_microsoft` is preferred and always available.

### Azure Blob Storage Access Tiers

Azure Blob Storage containers have three public access levels:

| Access Level | Container | Blobs | Risk |
|---|---|---|---|
| **Private** | No | No | None — requires SAS token or auth |
| **Blob** | No | Yes | Medium — can read individual blobs if URL known |
| **Container** | Yes | Yes | High — can list all blobs + read them |

BBOT's `bucket_microsoft` module detects all three tiers and flags Container-level as a HIGH finding.

### Scan Command (Azure Deep Dive)

```bash
bbot -t example.com \
     -m azure_realm azure_tenant bucket_microsoft \
     -c modules.bucket_microsoft.permutations=true \
     -om json web_report \
     -o ~/bug_bounty/acme/bbot_scans/ \
     -n acme_azure_deep
```

### Manual Verification After BBOT

```bash
# Test container-level public access (list all blobs)
ACCOUNT="companyaccount"
CONTAINER="public"

curl -s "https://${ACCOUNT}.blob.core.windows.net/${CONTAINER}?restype=container&comp=list" | \
     python3 -m xml.dom.minidom

# Test blob-level access (read specific file if URL known)
curl -sI "https://${ACCOUNT}.blob.core.windows.net/${CONTAINER}/config.json"

# Download a specific blob
curl -s "https://${ACCOUNT}.blob.core.windows.net/${CONTAINER}/config.json" -o config.json

# List containers in a storage account (if full public access)
curl -s "https://${ACCOUNT}.blob.core.windows.net/?comp=list" | python3 -m xml.dom.minidom
```

### Azure Storage URL Pattern

```
https://[storage-account].blob.core.windows.net/[container]/[blob-path]

Examples:
  https://acmecorp.blob.core.windows.net/
  https://acme-backup.blob.core.windows.net/data/
  https://acmeassets.blob.core.windows.net/images/logo.png
```

---

## bucket_firebase

**Description:** Discovers and tests Firebase Realtime Database and Cloud Firestore instances related to the target. Firebase databases with open access rules can expose entire application data.

**Flags:** `active`, `safe`, `cloud-enum`
**Watched Events:** `DNS_NAME`, `STORAGE_BUCKET`
**Produced Events:** `STORAGE_BUCKET`, `FINDING`

**Options:**
| Option | Default | Description |
|---|---|---|
| `permutations` | `False` | Generate Firebase project name permutations |

**Firebase Checks:**
- Database existence (`https://[project].firebaseio.com/`)
- Public read access (`.json` endpoint)
- Public write access
- Unauthenticated data exposure

**Firebase URL Pattern:**
`https://[project-id].firebaseio.com/.json`

**Usage Examples:**
```bash
bbot -t example.com -m bucket_firebase

# With permutations
bbot -t example.com -m bucket_firebase \
     -c modules.bucket_firebase.permutations=true
```

---

## bucket_digitalocean

**Description:** Discovers and tests DigitalOcean Spaces storage buckets. DigitalOcean Spaces is an S3-compatible object storage service.

**Flags:** `active`, `safe`, `cloud-enum`
**Watched Events:** `DNS_NAME`, `STORAGE_BUCKET`
**Produced Events:** `STORAGE_BUCKET`, `FINDING`

**Options:**
| Option | Default | Description |
|---|---|---|
| `permutations` | `False` | Generate Spaces bucket name permutations |

**Regions:** nyc3, sfo3, ams3, sgp1, fra1, syd1

**Usage Examples:**
```bash
bbot -t example.com -m bucket_digitalocean
```

---

## bucket_file_enum

**Description:** Downloads and enumerates files from publicly accessible storage buckets discovered by other bucket modules. Lists all accessible files and downloads sensitive ones for analysis.

**Flags:** `active`, `aggressive`
**Watched Events:** `STORAGE_BUCKET`
**Produced Events:** `FINDING`, `URL_UNVERIFIED`

**Options:**
| Option | Default | Description |
|---|---|---|
| `max_files` | `1000` | Maximum files to enumerate per bucket |
| `interesting_extensions` | Built-in list | File extensions to flag as interesting |

**Interesting Files Flagged:**
- `.env` files (environment variables)
- `.pem` / `.key` (private keys)
- `config.json` / `config.yml` (configuration files)
- `database.sql` / `backup.sql` (database dumps)
- `credentials.json` / `secrets.json`
- `*.pfx` / `*.p12` (certificate stores)
- `backup.zip` / `data.tar.gz` (archives)

**Usage Examples:**
```bash
# Full bucket discovery + file enumeration
bbot -t example.com \
     -m bucket_amazon bucket_google bucket_microsoft bucket_firebase \
        bucket_digitalocean bucket_file_enum \
     -n cloud_bucket_hunt
```

---

## azure_realm

**Description:** Checks Azure authentication realm information for a domain. Sends a request to Microsoft's authentication API to determine Azure AD tenant information, authentication methods, and federation configuration.

**Flags:** `passive`, `safe`, `cloud-enum`
**Watched Events:** `DNS_NAME`
**Produced Events:** `FINDING`

**Data Returned:**
- Azure AD tenant name
- Authentication type (Managed vs Federated)
- Federation endpoint (ADFS server URL)
- Cloud-only vs hybrid identity indicator
- Namespace type

**API Endpoint Used:**
`https://login.microsoftonline.com/getuserrealm.srf?login=user@domain.com&json=1`

**Usage Examples:**
```bash
bbot -t example.com -m azure_realm
```

**Why This Matters:**
- Reveals internal ADFS server URLs
- Indicates if target uses Azure AD
- Federation endpoint may be directly accessible
- Cloud-only organizations have different attack surface

---

## azure_tenant

**Description:** Enumerates Azure Active Directory tenant information. Discovers associated domains, tenant ID, and organizational metadata from Microsoft's public APIs.

**Flags:** `passive`, `safe`, `cloud-enum`
**Watched Events:** `DNS_NAME`
**Produced Events:** `FINDING`, `DNS_NAME`

**Information Retrieved:**
- Azure AD Tenant ID (UUID)
- Associated domains in the tenant
- Organization display name
- Tenant region scope

**Usage Examples:**
```bash
bbot -t example.com -m azure_realm azure_tenant
```

---

## censys_ip

**Description:** Queries Censys.io for IP address data. Returns service banners, open ports, SSL certificate information, and other metadata indexed by Censys's internet-wide scanning.

**Flags:** `passive`, `safe`, `cloud-enum`
**Watched Events:** `IP_ADDRESS`
**Produced Events:** `OPEN_TCP_PORT`, `PROTOCOL`, `FINDING`

**Options:**
| Option | Default | Description |
|---|---|---|
| `api_id` | `""` | Censys API ID |
| `api_secret` | `""` | Censys API Secret |

**Data Returned:**
- Open TCP/UDP ports
- Service banners and protocols
- SSL/TLS certificate details
- Autonomous System (AS) information
- Geographic location
- Reverse DNS records

**Usage Examples:**
```bash
bbot -t example.com -m censys_ip \
     -c modules.censys_ip.api_id=ID \
        modules.censys_ip.api_secret=SECRET
```

---

## shodan_idb

**Description:** Queries Shodan's InternetDB API — a free, no-auth-required API that provides cached scan data for IP addresses. Returns ports, vulnerabilities (CVEs), hostnames, and CPE software identifiers.

**Flags:** `passive`, `safe`
**Watched Events:** `IP_ADDRESS`
**Produced Events:** `OPEN_TCP_PORT`, `VULNERABILITY`, `FINDING`

**InternetDB API Features:**
- No API key required (free tier)
- Returns cached data (may be weeks old)
- CVEs associated with the IP
- Open ports
- Hostnames via reverse DNS
- CPE software identifiers

**Usage Examples:**
```bash
# Quick passive IP intelligence
bbot -t example.com -m shodan_idb

# With shodan_dns for subdomain data too
bbot -t example.com -m shodan_dns shodan_idb
```

**Note:** `shodan_idb` uses the free InternetDB endpoint (`internetdb.shodan.io`) vs `shodan_dns` which requires an API key.

---

## Cloud Enumeration Workflows

### Full Multi-Cloud Bucket Hunt
```bash
bbot -t example.com \
     -m bucket_amazon bucket_google bucket_microsoft \
        bucket_firebase bucket_digitalocean bucket_file_enum \
     -c modules.bucket_amazon.permutations=true \
        modules.bucket_google.permutations=true \
        modules.bucket_microsoft.permutations=true \
     -n full_cloud_bucket_hunt
```

### Azure Infrastructure Discovery
```bash
bbot -t example.com \
     -m azure_realm azure_tenant bucket_microsoft \
     -n azure_discovery
```

### Passive Cloud Intelligence
```bash
bbot -t example.com \
     -m azure_realm azure_tenant shodan_idb censys_ip \
     -c modules.censys_ip.api_id=ID \
        modules.censys_ip.api_secret=SECRET \
     -f passive \
     -n passive_cloud_intel
```

### Cloud Preset
```bash
bbot -t example.com -p cloud-enum
```

### Storage Bucket Targeting (Known Name)
```bash
# Target a specific known bucket
bbot -t "s3://company-backup" -m bucket_file_enum

# Generate permutations for a known prefix
bbot -t example.com -m bucket_amazon \
     -c modules.bucket_amazon.permutations=true \
     -n bucket_permutation_hunt
```

### Event Flow
```
DNS_NAME
    ├─> bucket_amazon → STORAGE_BUCKET → bucket_file_enum → FINDING
    ├─> bucket_google → STORAGE_BUCKET → bucket_file_enum → FINDING
    ├─> bucket_microsoft → STORAGE_BUCKET → bucket_file_enum → FINDING
    ├─> bucket_firebase → STORAGE_BUCKET → FINDING
    ├─> azure_realm → FINDING (tenant info)
    ├─> azure_tenant → FINDING, DNS_NAME (associated domains)
    └─> shodan_idb → OPEN_TCP_PORT, VULNERABILITY
```
