---
name: Email and Credential Discovery Modules
description: Complete reference for email enumeration, credential exposure, and authentication intelligence modules
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
# BBot Email & Credential Discovery Modules

Complete reference for email enumeration, credential exposure, and authentication intelligence modules.

---

## emailformat

**Description:** Queries emailformat.com to discover the email format used by the target organization (e.g., `firstname.lastname@company.com`, `flastname@company.com`). Understanding the email format enables targeted phishing, credential stuffing, and further enumeration.

**Flags:** `passive`, `safe`, `email-enum`
**Watched Events:** `DNS_NAME`
**Produced Events:** `EMAIL_ADDRESS`

**Data Returned:**
- Email format patterns
- Sample email addresses
- Alternative format patterns

```bash
bbot -t example.com -m emailformat
```

---

## hunterio

**Description:** Queries Hunter.io's email finding API. Hunter.io aggregates email addresses, email format patterns, and the sources where emails were found (websites, PDFs, public records).

**Flags:** `passive`, `safe`, `email-enum`, `subdomain-enum`
**Watched Events:** `DNS_NAME`
**Produced Events:** `EMAIL_ADDRESS`, `DNS_NAME`, `URL_UNVERIFIED`

**Options:**
| Option | Default | Description |
|---|---|---|
| `api_key` | `""` | Hunter.io API key (free tier: 25 searches/month) |

**Data Returned:**
- Verified email addresses
- Email format for the domain
- Sources where emails were discovered
- Department/role associations
- Name associated with email

```bash
bbot -t example.com -m hunterio \
     -c modules.hunterio.api_key=YOUR_HUNTER_KEY

# Combined email enumeration
bbot -t example.com \
     -m emailformat hunterio skymem pgp \
     -c modules.hunterio.api_key=KEY \
     -n email_enum
```

---

## skymem

**Description:** Queries skymem.info for email addresses associated with the target domain. Skymem aggregates email data from multiple public sources.

**Flags:** `passive`, `safe`, `email-enum`
**Watched Events:** `DNS_NAME`
**Produced Events:** `EMAIL_ADDRESS`

**No API key required.**

```bash
bbot -t example.com -m skymem
```

---

## pgp

**Description:** Queries public PGP/GPG keyservers for cryptographic keys associated with the target domain. PGP keys contain email addresses of employees who use encrypted email, often including executives and security teams.

**Flags:** `passive`, `safe`, `email-enum`
**Watched Events:** `DNS_NAME`
**Produced Events:** `EMAIL_ADDRESS`

**Keyservers Queried:**
- `keys.openpgp.org`
- `keyserver.ubuntu.com`
- `pgp.mit.edu`
- `keys.mailvelope.com`

```bash
bbot -t example.com -m pgp
```

**What PGP Keys Reveal:**
- Email addresses of employees
- Name associations
- When keys were created (tenure indicator)
- Key signatures (trust network / org relationships)

---

## dehashed

**Description:** Queries Dehashed.com — a database of leaked credential data. Searches for email addresses, usernames, and passwords associated with the target domain from known data breaches. Requires a paid subscription.

**Flags:** `passive`, `safe`, `email-enum`
**Watched Events:** `DNS_NAME`, `EMAIL_ADDRESS`
**Produced Events:** `FINDING`, `EMAIL_ADDRESS`

**Options:**
| Option | Default | Description |
|---|---|---|
| `api_key` | `""` | Dehashed API key |
| `username` | `""` | Dehashed account username (email) |

**Data Returned:**
- Leaked email addresses
- Associated usernames
- Hashed or plaintext passwords
- Source breach name
- IP addresses associated with accounts

```bash
bbot -t example.com -m dehashed \
     -c modules.dehashed.api_key=KEY \
        modules.dehashed.username=your@email.com
```

**Use Cases:**
- Identify compromised employee accounts
- Find password patterns for password spray
- Discover old/forgotten accounts
- Map internal usernames

---

## credshed

**Description:** Queries a self-hosted Credshed server for known credentials. Credshed is an open-source credential database aggregator. If the target organization runs Credshed or you have access to one, this module queries it for target domain credentials.

**Flags:** `passive`, `safe`
**Watched Events:** `DNS_NAME`
**Produced Events:** `FINDING`

**Options:**
| Option | Default | Description |
|---|---|---|
| `username` | `""` | Credshed server username |
| `password` | `""` | Credshed server password |
| `credshed_url` | `""` | URL of your Credshed instance |

```bash
bbot -t example.com -m credshed \
     -c modules.credshed.credshed_url=http://credshed.internal:8080 \
        modules.credshed.username=admin \
        modules.credshed.password=password
```

---

## trufflehog

**Description:** Scans discovered code repositories for leaked credentials using TruffleHog. Detects API keys, OAuth tokens, AWS credentials, private keys, and 700+ other secret types using regex patterns and entropy analysis.

**Flags:** `active`, `safe`, `code-enum`
**Watched Events:** `CODE_REPOSITORY`
**Produced Events:** `FINDING`, `VULNERABILITY`

**Dependencies:** `trufflehog` binary (auto-downloaded)

**Detector Coverage (700+ detectors):**
- AWS Access Keys and Secret Keys
- GitHub Personal Access Tokens
- Google OAuth / Service Account keys
- Stripe API keys
- Slack Bot tokens
- Twilio credentials
- SendGrid API keys
- Mailgun API keys
- Azure Service Principal credentials
- GCP Service Account keys
- Heroku API keys
- NPM tokens
- PyPI tokens
- DockerHub credentials
- JWT tokens
- Private keys (RSA, EC, DSA)
- SSH keys

**Usage Examples:**
```bash
# Scan discovered GitHub repos
bbot -t example.com \
     -m github_org git_clone trufflehog \
     -c modules.github_org.api_key=ghp_TOKEN \
        modules.git_clone.api_key=ghp_TOKEN

# Full code + credential scan
bbot -t example.com \
     -m github_org github_codesearch postman trufflehog \
     -c modules.github_org.api_key=ghp_TOKEN \
        modules.github_codesearch.api_key=ghp_TOKEN
```

---

## passivetotal

**Description:** Queries the PassiveTotal (RiskIQ / Microsoft Defender) platform for passive DNS, WHOIS, certificates, and account intelligence data. Premium service with extensive historical data.

**Flags:** `passive`, `safe`, `subdomain-enum`
**Watched Events:** `DNS_NAME`, `IP_ADDRESS`
**Produced Events:** `DNS_NAME`, `FINDING`

**Options:**
| Option | Default | Description |
|---|---|---|
| `api_key` | `""` | PassiveTotal API key |
| `username` | `""` | PassiveTotal account email |

**Data Returned:**
- Historical DNS records
- WHOIS registration data
- SSL certificate history
- Associated IP addresses
- Malware/threat intelligence tags
- Connected domain names

```bash
bbot -t example.com -m passivetotal \
     -c modules.passivetotal.api_key=KEY \
        modules.passivetotal.username=user@email.com
```

---

## Email & Credential Workflows

### Full Email Enumeration (Preset)
```bash
bbot -t example.com -p email-enum \
     -c modules.hunterio.api_key=HUNTER_KEY
```

### Manual Full Email Enumeration
```bash
bbot -t example.com \
     -m emailformat hunterio skymem pgp \
     -c modules.hunterio.api_key=KEY \
     -n full_email_enum
```

### Breach Intelligence Gathering
```bash
bbot -t example.com \
     -m dehashed passivetotal emailformat hunterio \
     -c modules.dehashed.api_key=KEY \
        modules.dehashed.username=you@email.com \
        modules.passivetotal.api_key=KEY \
        modules.passivetotal.username=you@email.com \
        modules.hunterio.api_key=KEY \
     -n breach_intel
```

### Credential Leak Hunt in Code
```bash
bbot -t example.com \
     -m github_org github_codesearch postman postman_download trufflehog \
     -c modules.github_org.api_key=ghp_TOKEN \
        modules.github_codesearch.api_key=ghp_TOKEN \
        modules.postman.api_key=PMAK_TOKEN \
     -n credential_leak_hunt
```

### Complete People Intelligence
```bash
bbot -t example.com \
     -m emailformat hunterio skymem pgp \
        github_usersearch social \
     -c modules.hunterio.api_key=HUNTER_KEY \
        modules.github_usersearch.api_key=ghp_TOKEN \
     -n people_intel
```

### Event Flow
```
DNS_NAME
    ├─> emailformat → EMAIL_ADDRESS (format patterns)
    ├─> hunterio → EMAIL_ADDRESS, DNS_NAME (verified emails)
    ├─> skymem → EMAIL_ADDRESS (scraped emails)
    ├─> pgp → EMAIL_ADDRESS (key owners)
    ├─> dehashed → FINDING (leaked credentials)
    └─> passivetotal → DNS_NAME, FINDING (passive DNS history)

CODE_REPOSITORY (from github_org, postman, etc.)
    └─> trufflehog → FINDING (hardcoded secrets)
```
