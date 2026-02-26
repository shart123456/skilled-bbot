---
name: Code Leak and Secret Hunt
description: Discover leaked source code and hardcoded secrets via GitHub, GitLab, Postman, and Trufflehog
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
# Workflow: Code Leak & Secret Hunt

**Risk Level:** Low — passive queries to third-party services and repos
**Use When:** Software companies, SaaS products, developer-heavy organizations

---

## Overview

Code leak hunting discovers publicly exposed source code, credentials, API keys, and internal infrastructure details through GitHub, GitLab, Postman, Docker, and mobile apps. Findings here often lead directly to critical vulnerabilities.

**Modules Used:** github_org, github_codesearch, github_usersearch, github_workflows, gitlab_com, gitlab_onprem, postman, postman_download, git, gitdumper, dockerhub, docker_pull, trufflehog, code_repository, httpx, dehashed, credshed

---

## Phase 1: GitHub Organization Intelligence

```bash
COMPANY="acme"
TARGET="acme.com"
GITHUB_TOKEN="ghp_YourTokenHere"

# Discover all org repos and member repos
bbot -t $TARGET \
     -m github_org github_workflows \
     -c modules.github_org.api_key=$GITHUB_TOKEN \
        modules.github_org.include_members=true \
        modules.github_org.include_member_repos=true \
        modules.github_workflows.api_key=$GITHUB_TOKEN \
     -om json \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_github_org
```

---

## Phase 2: GitHub Code Search

```bash
# Search all public GitHub code mentioning the domain
bbot -t $TARGET \
     -m github_codesearch \
     -c modules.github_codesearch.api_key=$GITHUB_TOKEN \
        modules.github_codesearch.limit=500 \
     -om json \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_github_search
```

---

## Phase 3: Postman Intelligence

```bash
# Find public Postman collections with company API docs
bbot -t $TARGET \
     -m postman postman_download \
     -c modules.postman.api_key=$POSTMAN_KEY \
     -om json \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_postman
```

---

## Phase 4: Exposed Git Directories

```bash
# Check web servers for exposed .git/
bbot -t $TARGET \
     -m httpx git gitdumper \
     -om json \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_git_exposure
```

---

## Phase 5: Credential Scanning with TruffleHog

```bash
# Scan all discovered repos for hardcoded secrets
bbot -t $TARGET \
     -m github_org git_clone trufflehog \
     -c modules.github_org.api_key=$GITHUB_TOKEN \
        modules.git_clone.api_key=$GITHUB_TOKEN \
        modules.git_clone.output_folder=/opt/repos/$COMPANY/ \
     -om json \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_trufflehog
```

---

## Phase 6: Docker Intelligence

```bash
# Find and analyze Docker images
bbot -t $TARGET \
     -m dockerhub docker_pull \
     -om json \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_docker
```

---

## Phase 6B: On-Premises GitLab Detection

Many organizations run self-hosted GitLab instances that may be accessible from the internet:

```bash
# Detect self-hosted GitLab instances
bbot -t $TARGET \
     -m httpx gitlab_onprem \
     -om json \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_gitlab_onprem
```

**What `gitlab_onprem` does:**
- Watches `URL` and `HTTP_RESPONSE` events from `httpx`
- Fingerprints responses for GitLab self-hosted indicators (meta tags, headers, `/users/sign_in` page)
- Produces `FINDING` events for confirmed GitLab instances
- May also identify public/private registration status

**What to look for:**
```bash
LATEST=$(ls -td ~/bug_bounty/$COMPANY/bbot_scans/*/ | head -1)
jq 'select(.module=="gitlab_onprem") | select(.type=="FINDING")' $LATEST/output.ndjson
```

**Follow-up manual checks:**
```bash
# Test registration open (user enumeration)
curl -s https://gitlab.example.com/api/v4/users/1 | jq '.username'

# Check for public repos
curl -s "https://gitlab.example.com/api/v4/projects?visibility=public&per_page=20" | \
     jq '.[].web_url'

# Test sign-up available
curl -sI "https://gitlab.example.com/users/sign_up" | grep "HTTP/"
```

---

## Phase 7: Credential Database Search

Search leaked credential databases for target employee credentials:

### Option A: Dehashed (Paid API)

```bash
DEHASHED_EMAIL="your@email.com"
DEHASHED_KEY="your_dehashed_api_key"

# Search by domain
bbot -t $TARGET \
     -m dehashed \
     -c modules.dehashed.username=$DEHASHED_EMAIL \
        modules.dehashed.api_key=$DEHASHED_KEY \
     -om json \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_dehashed
```

**Analyze dehashed results:**
```bash
LATEST=$(ls -td ~/bug_bounty/$COMPANY/bbot_scans/*/ | head -1)

# All credential findings
jq 'select(.module=="dehashed") | select(.type=="FINDING")' $LATEST/output.ndjson

# Email/password pairs
jq -r 'select(.module=="dehashed") | select(.type=="FINDING") | .data' \
   $LATEST/output.ndjson | head -20
```

### Option B: Credshed (Self-Hosted)

If you run a self-hosted Credshed instance with a credential database:

```bash
CREDSHED_URL="http://localhost:8080"
CREDSHED_USER="admin"
CREDSHED_PASS="yourpassword"

bbot -t $TARGET \
     -m credshed \
     -c modules.credshed.url=$CREDSHED_URL \
        modules.credshed.username=$CREDSHED_USER \
        modules.credshed.password=$CREDSHED_PASS \
     -om json \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_credshed
```

**Analyze credshed results:**
```bash
LATEST=$(ls -td ~/bug_bounty/$COMPANY/bbot_scans/*/ | head -1)
jq 'select(.module=="credshed") | select(.type=="CREDENTIAL" or .type=="FINDING")' \
   $LATEST/output.ndjson
```

> **See also:** `workflows/mobile_app_recon.md` — APK decompilation often surfaces hardcoded credentials that can be cross-referenced against leaked credential databases.

---

## Full Code Enum Preset

```bash
bbot -t $TARGET -p code-enum \
     -c modules.github_org.api_key=$GITHUB_TOKEN \
        modules.github_codesearch.api_key=$GITHUB_TOKEN \
        modules.postman.api_key=$POSTMAN_KEY \
     -om json web_report \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_code_preset
```

---

## All-In Code Leak Hunt

```bash
bbot -t $TARGET \
     -m github_org github_codesearch github_usersearch \
        github_workflows gitlab_com \
        postman postman_download \
        dockerhub \
        httpx git gitdumper code_repository \
        trufflehog \
     -c modules.github_org.api_key=$GITHUB_TOKEN \
        modules.github_org.include_members=true \
        modules.github_org.include_member_repos=true \
        modules.github_codesearch.api_key=$GITHUB_TOKEN \
        modules.github_codesearch.limit=500 \
        modules.github_usersearch.api_key=$GITHUB_TOKEN \
        modules.github_workflows.api_key=$GITHUB_TOKEN \
        modules.gitlab_com.api_key=$GITLAB_TOKEN \
        modules.postman.api_key=$POSTMAN_KEY \
     -om json web_report \
     -o ~/bug_bounty/$COMPANY/bbot_scans/ \
     -n ${COMPANY}_full_code_hunt
```

---

## Analyzing Code Leak Results

```bash
LATEST=$(ls -td ~/bug_bounty/$COMPANY/bbot_scans/*/ | head -1)

# All code repositories found
echo "=== Code Repositories ==="
jq -r 'select(.type=="CODE_REPOSITORY") | .data' $LATEST/output.ndjson | sort -u

# TruffleHog secrets
echo "=== Secrets Found (TruffleHog) ==="
jq 'select(.module=="trufflehog") | select(.type=="FINDING" or .type=="VULNERABILITY")' \
   $LATEST/output.ndjson

# Exposed git directories
echo "=== Exposed .git Directories ==="
jq -r 'select(.module=="git") | select(.type=="FINDING") | .data' $LATEST/output.ndjson

# GitHub workflow secrets in env vars
echo "=== Workflow Files ==="
jq 'select(.module=="github_workflows") | select(.type=="FINDING")' $LATEST/output.ndjson
```

---

## Manual Investigation After BBOT

### Clone and Search Discovered Repos

```bash
# After github_org discovers repos, clone them
cd /opt/repos/$COMPANY/

# Run trufflehog manually on all cloned repos
find . -name ".git" -type d | while read gitdir; do
  repo=$(dirname "$gitdir")
  echo "Scanning: $repo"
  trufflehog git file://$repo --only-verified 2>/dev/null
done

# Search for hardcoded strings
grep -r "password\|secret\|api_key\|token\|credential" \
     --include="*.py" --include="*.js" --include="*.json" \
     --include="*.yml" --include="*.env" \
     . | grep -v ".git" | grep -v "node_modules"
```

### Analyze Postman Collections

```bash
# Find API keys in downloaded Postman collections
find ~/bug_bounty/$COMPANY/bbot_scans/ -name "*.json" | \
  xargs grep -l "Authorization\|api_key\|token\|Bearer" 2>/dev/null | \
  xargs cat | jq '.variable[]? | select(.value != "" and .value != null)'
```

### Check GitHub Actions for Secrets

```bash
# Look for hardcoded values in CI files
find /opt/repos/$COMPANY/ -name "*.yml" -path "*/.github/workflows/*" | \
  xargs grep -l "AWS\|SECRET\|TOKEN\|KEY\|PASSWORD" | head -20

# Check for environment variable exposure
find /opt/repos/$COMPANY/ -name "*.yml" -path "*workflows*" -exec \
  grep -Hn "env:" {} \;
```

---

## High-Value Secret Patterns to Search

```bash
# AWS credentials
grep -r "AKIA[0-9A-Z]{16}" /opt/repos/$COMPANY/ --include="*.py" --include="*.js" --include="*.env"

# GitHub tokens
grep -r "ghp_[a-zA-Z0-9]{36}" /opt/repos/$COMPANY/

# API keys in config files
grep -rn "api_key\s*=\s*['\"][a-zA-Z0-9]{20,}" /opt/repos/$COMPANY/ --include="*.json"

# Private keys
grep -rn "BEGIN RSA PRIVATE KEY\|BEGIN EC PRIVATE KEY\|BEGIN PRIVATE KEY" /opt/repos/$COMPANY/

# Database URLs
grep -rn "postgresql://\|mysql://\|mongodb+srv://" /opt/repos/$COMPANY/
```
