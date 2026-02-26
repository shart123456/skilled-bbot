---
name: Mobile App Reconnaissance
description: Discover Android APKs via Google Play Store, decompile with jadx, and perform mobile app OSINT with BeVigil
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
# Workflow: Mobile App Reconnaissance

**Risk Level:** Low — queries public app stores and mobile OSINT sources only
**Use When:** Target has a mobile app (Android/iOS), bug bounty scope includes mobile, or mobile endpoints are in scope

---

## Overview

Mobile apps are gold mines for bug bounty hunters — they often contain hardcoded API endpoints, staging environment URLs, authentication tokens, and secrets that backend engineers forget to remove. This workflow discovers apps, downloads APKs, decompiles them, and uses OSINT services to find extracted secrets.

**Modules Used:** `google_playstore`, `apkpure`, `jadx`, `bevigil`

---

## Prerequisites

```bash
# Install jadx for APK decompilation
sudo apt install jadx

# Verify installation
jadx --version
```

---

## Phase 1: App Discovery (Google Play Store)

Find all apps published by the target organization:

```bash
COMPANY="acme"
TARGET="acme.com"

bbot -t $TARGET \
     -m google_playstore \
     -om json \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_playstore_discovery
```

**What google_playstore does:**
- Searches Google Play Store for apps matching the target domain/company name
- Produces `CODE_REPOSITORY` events pointing to Play Store app pages
- No API key required — uses public Play Store search API

**Expected output:**
```bash
LATEST=$(ls -td ~/bug_bounty/$COMPANY/bbot_scans/*/ | head -1)

# View discovered apps
jq -r 'select(.type=="CODE_REPOSITORY") | select(.module=="google_playstore") | .data' \
   $LATEST/output.ndjson
```

---

## Phase 2: APK Download (apkpure)

Download APKs from apkpure.com for offline analysis:

```bash
# apkpure watches CODE_REPOSITORY events from google_playstore
bbot -t $TARGET \
     -m google_playstore apkpure \
     -om json \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_apk_download
```

**What apkpure does:**
- Watches `CODE_REPOSITORY` events (from `google_playstore`)
- Downloads the APK from apkpure.com (third-party APK mirror)
- Produces `FILESYSTEM` events pointing to downloaded APK files
- No API key required

**Find downloaded APKs:**
```bash
LATEST=$(ls -td ~/bug_bounty/$COMPANY/bbot_scans/*/ | head -1)
jq -r 'select(.module=="apkpure") | select(.type=="FILESYSTEM") | .data' \
   $LATEST/output.ndjson
```

---

## Phase 3: APK Decompilation (jadx)

Decompile downloaded APKs to Java/smali source:

```bash
# jadx watches CODE_REPOSITORY events from apkpure
# Prerequisite: apt install jadx
bbot -t $TARGET \
     -m google_playstore apkpure jadx \
     -om json \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_apk_decompile
```

**What jadx does:**
- Watches `CODE_REPOSITORY` events produced by `apkpure`
- Runs jadx decompiler on each downloaded APK
- Produces `FILESYSTEM` events for the decompiled source tree
- **Prerequisite:** `jadx` must be installed (`apt install jadx`)

**Find decompiled source:**
```bash
LATEST=$(ls -td ~/bug_bounty/$COMPANY/bbot_scans/*/ | head -1)
DECOMPILE_DIR=$(jq -r 'select(.module=="jadx") | select(.type=="FILESYSTEM") | .data' \
                $LATEST/output.ndjson | head -1)
echo "Decompiled source at: $DECOMPILE_DIR"
```

---

## Phase 4: BeVigil OSINT (Passive Alternative)

Query BeVigil's mobile app security database — no APK download needed:

```bash
# bevigil requires API key (free tier available at bevigil.com)
BEVIGIL_KEY="your_bevigil_key"

bbot -t $TARGET \
     -m bevigil \
     -c modules.bevigil.api_key=$BEVIGIL_KEY \
     -om json \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_bevigil
```

**What bevigil does:**
- Queries BeVigil's database of pre-analyzed mobile apps
- Returns endpoints, secrets, and URLs extracted from APKs
- Passive — no direct contact with target
- Produces `URL_UNVERIFIED` and `FINDING` events

**BeVigil results:**
```bash
LATEST=$(ls -td ~/bug_bounty/$COMPANY/bbot_scans/*/ | head -1)

# URLs found in mobile apps
jq -r 'select(.module=="bevigil") | select(.type=="URL_UNVERIFIED") | .data' \
   $LATEST/output.ndjson | sort -u

# Findings (secrets, endpoints)
jq 'select(.module=="bevigil") | select(.type=="FINDING")' \
   $LATEST/output.ndjson
```

---

## Phase 5: Full Mobile Recon (All Modules)

Consolidated command running all four modules:

```bash
COMPANY="acme"
TARGET="acme.com"
BEVIGIL_KEY="your_bevigil_key"

bbot -t $TARGET \
     -m google_playstore apkpure jadx bevigil \
     -c modules.bevigil.api_key=$BEVIGIL_KEY \
     -om json \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_mobile_full
```

---

## Manual Investigation After BBOT

### Search Decompiled Source for Secrets

```bash
DECOMPILE_DIR="/path/to/decompiled/app"

# API keys and secrets
grep -r "api_key\|apikey\|api_secret\|secret_key\|SECRET" \
     --include="*.java" --include="*.kt" --include="*.xml" \
     $DECOMPILE_DIR | grep -v ".class" | head -50

# Hardcoded URLs and base URLs
grep -r "BASE_URL\|baseUrl\|base_url\|https://\|http://" \
     --include="*.java" --include="*.kt" --include="*.xml" \
     $DECOMPILE_DIR | grep -v ".class" | grep -v "android\." | head -50

# Authentication tokens
grep -r "Bearer\|Authorization\|token\|password\|passwd\|credential" \
     --include="*.java" --include="*.kt" \
     $DECOMPILE_DIR | grep -v ".class" | head -50

# AWS credentials
grep -r "AKIA[0-9A-Z]\{16\}" $DECOMPILE_DIR

# Private keys
grep -r "BEGIN RSA PRIVATE\|BEGIN EC PRIVATE\|BEGIN PRIVATE KEY" $DECOMPILE_DIR
```

### Extract and Test API Endpoints

```bash
# Extract all unique URLs from decompiled source
grep -roh 'https://[a-zA-Z0-9./_?=&%-]*' $DECOMPILE_DIR \
     --include="*.java" --include="*.kt" | \
     grep -v "android\.\|google\.\|schemas\." | \
     sort -u > /tmp/mobile_endpoints.txt

echo "Found $(wc -l < /tmp/mobile_endpoints.txt) unique endpoints"
cat /tmp/mobile_endpoints.txt
```

### Check AndroidManifest.xml

```bash
# Permissions and exported components
cat $DECOMPILE_DIR/resources/AndroidManifest.xml | \
    grep -E "android:exported|android:permission|uses-permission" | head -30

# Deep link handlers (attack surface)
cat $DECOMPILE_DIR/resources/AndroidManifest.xml | \
    grep -A5 "intent-filter" | grep -E "scheme|host|path" | head -20
```

---

## Result Analysis

```bash
LATEST=$(ls -td ~/bug_bounty/$COMPANY/bbot_scans/*/ | head -1)

# Apps discovered
echo "=== Apps Found ==="
jq -r 'select(.type=="CODE_REPOSITORY") | select(.module=="google_playstore") | .data' \
   $LATEST/output.ndjson

# APKs downloaded
echo ""
echo "=== APKs Downloaded ==="
jq -r 'select(.module=="apkpure") | select(.type=="FILESYSTEM") | .data' \
   $LATEST/output.ndjson

# BeVigil URLs
echo ""
echo "=== BeVigil Endpoints ==="
jq -r 'select(.module=="bevigil") | select(.type=="URL_UNVERIFIED") | .data' \
   $LATEST/output.ndjson | sort -u

# All findings
echo ""
echo "=== Findings ==="
jq 'select(.type=="FINDING")' $LATEST/output.ndjson
```

---

## Memory Integration

**At scan start — query prior intel:**
```
Tool: qdrant-find
Collection: target_intel
Query: "mobile app endpoints API keys [TARGET]"
```

**After scan — store mobile findings:**
```
Tool: qdrant-store
Collection: scan_findings
Content: Mobile recon on [TARGET]: found [N] apps, [N] endpoints, [N] secrets
Metadata: {
  "target": "[TARGET]",
  "finding_type": "mobile_app",
  "data": "[apps found, endpoints extracted]",
  "severity": "info",
  "source_module": "mobile_app_recon_workflow",
  "scan_type": "mobile_recon",
  "timestamp": "[ISO 8601]"
}
```

**Store individual secret findings:**
```
Tool: qdrant-store
Collection: scan_findings
Content: Hardcoded [secret_type] found in [app_name] APK at [location]
Metadata: {
  "target": "[TARGET]",
  "finding_type": "secret",
  "severity": "high",
  "source_module": "jadx_manual",
  "timestamp": "[ISO 8601]"
}
```

---

## Safety Checklist

- [ ] App store queries only — no direct contact with target infrastructure
- [ ] APK downloads from apkpure (third-party mirror), not target servers
- [ ] BeVigil is fully passive — queries its own database
- [ ] Manual jadx decompilation is local — no network requests
- [ ] Output directory set outside public repos
- [ ] API keys not hardcoded in scripts

---

## Cross-References

- **BeVigil module doc:** `documentation/modules_ip_intelligence.md` (BeVigil section)
- **Code leak hunt:** `workflows/code_leak_hunt.md` — mobile APKs often contain GitHub links and API keys that chain into code leak findings
- **Web app audit:** `workflows/web_app_audit.md` — test extracted mobile API endpoints with paramminer/generic_ssrf
