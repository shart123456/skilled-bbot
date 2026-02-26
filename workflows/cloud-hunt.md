# Workflow: Cloud Asset Discovery & Bucket Hunt

**Risk Level:** Low — checking public access, not exploiting
**Use When:** Target uses AWS/GCP/Azure, cloud-heavy companies, SaaS products

---

## Overview

Cloud enumeration discovers storage buckets, cloud services, infrastructure, and tenant information for cloud-heavy targets. Misconfigured S3 buckets, Azure Blob Storage, and Firebase databases are among the most commonly reported high-severity findings.

**Modules Used:** bucket_amazon, bucket_google, bucket_microsoft, bucket_firebase, bucket_digitalocean, bucket_file_enum, azure_realm, azure_tenant, shodan_idb, censys_ip

---

## Phase 1: Azure Infrastructure Discovery

```bash
COMPANY="acme"
TARGET="acme.com"

# Azure tenant & realm identification
bbot -t $TARGET \
     -m azure_realm azure_tenant \
     -om json \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_azure_intel

# Extract Azure info
LATEST=$(ls -td ~/bug_bounty/$COMPANY/bbot_scans/*/ | head -1)
jq 'select(.module=="azure_realm" or .module=="azure_tenant")' \
   $LATEST/output.ndjson
```

**What to Look For:**
- Federated authentication → ADFS server URL exposed
- Tenant ID → required for further Azure enumeration
- Associated domains → additional targets
- Cloud-only vs hybrid → different attack paths

---

## Phase 2: Multi-Cloud Bucket Hunt (No Permutations)

```bash
# Fast scan — known name patterns only
bbot -t $TARGET \
     -m bucket_amazon bucket_google bucket_microsoft \
        bucket_firebase bucket_digitalocean \
     -om json web_report \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_bucket_basic
```

---

## Phase 3: Permutation-Based Bucket Hunt

More thorough — generates many bucket name variations:

```bash
# Enable permutations for maximum coverage
bbot -t $TARGET \
     -m bucket_amazon bucket_google bucket_microsoft \
        bucket_firebase bucket_digitalocean \
     -c modules.bucket_amazon.permutations=true \
        modules.bucket_google.permutations=true \
        modules.bucket_microsoft.permutations=true \
        modules.bucket_firebase.permutations=true \
        modules.bucket_digitalocean.permutations=true \
     -om json web_report \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_bucket_permutation
```

---

## Phase 4: File Enumeration from Open Buckets

When open buckets are found:

```bash
# Auto-enumerate files from all open buckets
bbot -t $TARGET \
     -m bucket_amazon bucket_google bucket_microsoft \
        bucket_firebase bucket_digitalocean bucket_file_enum \
     -c modules.bucket_amazon.permutations=true \
        modules.bucket_google.permutations=true \
        modules.bucket_microsoft.permutations=true \
     -om json web_report \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_bucket_full
```

---

## Phase 5: Passive IP Intelligence

Identify what cloud services target IPs are on:

```bash
# Shodan InternetDB (free, no key needed)
bbot -t $TARGET \
     -m shodan_idb \
     -om json \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_ip_intel

# With Censys (paid, more detail)
bbot -t $TARGET \
     -m censys_ip censys_dns \
     -c modules.censys_ip.api_id=$CENSYS_ID \
        modules.censys_ip.api_secret=$CENSYS_SECRET \
        modules.censys_dns.api_id=$CENSYS_ID \
        modules.censys_dns.api_secret=$CENSYS_SECRET \
     -om json \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_censys_intel
```

---

## Full Cloud Preset

```bash
bbot -t $TARGET -p cloud-enum \
     -c modules.bucket_amazon.permutations=true \
        modules.bucket_google.permutations=true \
        modules.bucket_microsoft.permutations=true \
     -om json web_report \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_cloud_preset
```

---

## Analyzing Cloud Results

```bash
LATEST=$(ls -td ~/bug_bounty/$COMPANY/bbot_scans/*/ | head -1)

# All storage buckets found
echo "=== Storage Buckets ==="
jq -r 'select(.type=="STORAGE_BUCKET") | "\(.data.name) [\(.data.url)]"' \
   $LATEST/output.ndjson

# Open buckets (public access)
echo "=== OPEN Buckets ==="
jq -r 'select(.type=="FINDING") |
  select(.data | contains("open") or contains("public") or contains("listable")) |
  .data' \
  $LATEST/output.ndjson

# Files found in open buckets
echo "=== Files in Open Buckets ==="
jq -r 'select(.module=="bucket_file_enum") |
  select(.type=="FINDING") | .data' \
  $LATEST/output.ndjson

# Azure findings
echo "=== Azure Intelligence ==="
jq -r 'select(.module | startswith("azure")) | "\(.module): \(.data)"' \
   $LATEST/output.ndjson
```

---

## Manual Bucket Verification

After BBOT identifies buckets:

```bash
# Verify S3 bucket is open (list access)
aws s3 ls s3://company-bucket --no-sign-request

# Download files from open S3 bucket
aws s3 sync s3://company-bucket ./local-copy --no-sign-request

# Check Azure blob (public)
curl -sI "https://companyaccount.blob.core.windows.net/container?restype=container&comp=list"

# Check Firebase
curl -s "https://company-app.firebaseio.com/.json?shallow=true"

# Check GCS
curl -sI "https://storage.googleapis.com/company-bucket/"
```

---

## Cloud-Specific Target Lists

When you know the cloud provider:

```bash
# AWS-heavy company
bbot -t $TARGET \
     -m bucket_amazon azure_realm shodan_idb \
     -c modules.bucket_amazon.permutations=true \
     -n aws_heavy

# Azure-heavy company
bbot -t $TARGET \
     -m azure_realm azure_tenant bucket_microsoft \
     -c modules.bucket_microsoft.permutations=true \
     -n azure_heavy

# GCP-heavy company
bbot -t $TARGET \
     -m bucket_google bucket_firebase \
     -c modules.bucket_google.permutations=true \
        modules.bucket_firebase.permutations=true \
     -n gcp_heavy

# Multi-cloud
bbot -t $TARGET \
     -f cloud-enum \
     -c modules.bucket_amazon.permutations=true \
        modules.bucket_google.permutations=true \
        modules.bucket_microsoft.permutations=true \
     -n multi_cloud
```

---

## Impact Classification for Cloud Findings

| Finding | Severity | Impact |
|---|---|---|
| Open S3 bucket (list + read) | HIGH | Data exposure, information disclosure |
| Open S3 bucket (write) | CRITICAL | Data tampering, malware hosting |
| Open Firebase database | HIGH/CRITICAL | Full data access, potential write |
| Azure blob public access (full) | HIGH | Data exposure |
| Azure blob public access (read) | MEDIUM | Information disclosure |
| GCS public bucket | HIGH | Data exposure |
| Azure ADFS server URL | LOW/INFO | Attack surface enumeration |
| Firebase database (no data) | LOW | Empty exposure |
