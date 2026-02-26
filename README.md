# skilled-bbot

A Claude Code skill for running BBOT (Bighuge BLS OSINT Tool) scans during bug bounty reconnaissance.

## What it does

Provides Claude with full context to execute intelligent BBOT scans that stay within program scope, respect rate limits, and select appropriate modules based on the engagement type (passive, safe-active, or thorough).

Covers all 124 BBOT modules organized by category:

- DNS and subdomain enumeration
- Port scanning and service detection
- Web scanning and fuzzing
- Vulnerability scanning
- Cloud storage enumeration (AWS, GCP, Azure, Firebase, DigitalOcean)
- Mobile app recon (Play Store, APK decompilation, BeVigil)
- Code repository discovery
- Email and credential discovery
- IP intelligence and technology detection
- Internal engine modules reference

## Structure

```
bbot/
├── SKILL.md                                  - Skill definition, triggers, workflow routing
├── configs/                                  - API key templates, custom preset YAMLs
├── data/                                     - Wordlists, target scope files
├── documentation/
│   ├── bbot_technical_reference.md           - Full BBOT config and architecture reference
│   ├── modules_cloud_enumeration.md          - Cloud and storage modules (incl. Azure deep dive)
│   ├── modules_code_repository.md            - GitHub, GitLab, Postman, Docker modules
│   ├── modules_dns_subdomain_enum.md         - DNS and subdomain enumeration modules
│   ├── modules_email_credentials.md          - Email and credential discovery modules
│   ├── modules_internal.md                   - Internal engine modules (aggregate, speculate, etc.)
│   ├── modules_ip_intelligence.md            - IP intelligence and tech detection modules
│   ├── modules_output_modules.md             - Output and reporting modules
│   ├── modules_port_scanning.md              - Port scanning and service detection modules
│   ├── modules_vulnerability_scanning.md     - Vulnerability scanning modules
│   └── modules_web_scanning.md               - Web scanning and fuzzing modules
├── examples/                                 - Sample execution flows
├── playbooks/                                - Multi-workflow orchestration guides
├── cauldron/                                 - Runtime scan artifacts and output
├── resources/                                - Helper scripts and API integration docs
├── templates/                                - Scan report output templates
└── workflows/
    ├── passive_recon.md                      - Zero-contact recon, API sources only
    ├── safe_active_recon.md                  - Light active scanning, safe for most programs
    ├── full_engagement.md                    - Comprehensive 8-phase engagement
    ├── subdomain_takeover.md                 - Dangling DNS and zone takeover detection
    ├── vuln_scan.md                          - Nuclei + badsecrets + baddns
    ├── cloud_hunt.md                         - S3, GCS, Azure blob enumeration
    ├── web_app_audit.md                      - Parameter mining, fuzzing, secret detection
    ├── code_leak_hunt.md                     - GitHub, GitLab, Postman, Trufflehog
    └── mobile_app_recon.md                   - Play Store, APK download, jadx, BeVigil
```

## Workflows

| Workflow | Risk | Description |
|---|---|---|
| `passive_recon` | Zero | No direct target interaction — API and CT sources only |
| `safe_active_recon` | Low | Light active scanning, safe for most bug bounty programs |
| `full_engagement` | Medium | Comprehensive 8-phase scan for permissive scopes |
| `subdomain_takeover` | Low | Detect dangling DNS and takeover candidates |
| `vuln_scan` | Medium | Nuclei + badsecrets + baddns on live hosts |
| `cloud_hunt` | Low | S3, GCS, Azure blob, Firebase enumeration |
| `web_app_audit` | Medium | Parameter mining, lightfuzz, secret detection, web fuzzing |
| `code_leak_hunt` | Low | GitHub, GitLab, Postman, Trufflehog, dehashed, credshed |
| `mobile_app_recon` | Low | Play Store, APK download, jadx decompilation, BeVigil OSINT |

## Requirements

- BBOT v2.8.2+ installed
- Claude Code with skills support
- `jadx` for mobile APK decompilation (`apt install jadx`)

## Setup After Cloning

**1. Clone the repo**

```bash
git clone git@github.com:shart123456/skilled-bbot.git
```

**2. Place it in your Claude Code skills directory**

```bash
mv claude-bbot-skill ~/.claude/skills/bbot
```

**3. Verify Claude Code picks it up**

```bash
ls ~/.claude/skills/bbot/SKILL.md
```

Claude Code automatically loads skills from `~/.claude/skills/`. No additional configuration required.

**4. Set up API keys (optional but recommended)**

Add these to your shell profile (`~/.bashrc` or `~/.zshrc`):

```bash
# Free keys
export GITHUB_TOKEN="ghp_..."
export VT_KEY="..."                # VirusTotal
export CHAOS_KEY="..."             # ProjectDiscovery Chaos
export HUNTER_KEY="..."            # Hunter.io
export POSTMAN_KEY="..."
export WPSCAN_KEY="..."
export BEVIGIL_KEY="..."           # BeVigil (mobile OSINT)
export LEAKIX_KEY="..."            # LeakIX

# Paid keys
export SHODAN_KEY="..."
export ST_KEY="..."                # SecurityTrails
export CENSYS_ID="..."
export CENSYS_SECRET="..."
export ZOOMEYE_KEY="..."           # ZoomEye
export C99_KEY="..."               # C99.nl
export FULLHUNT_KEY="..."
export PT_KEY="..."                # PassiveTotal
export PT_USER="..."               # PassiveTotal username
export TRICKEST_KEY="..."
export DEHASHED_KEY="..."          # Dehashed credential DB
```

Free API keys available at: GitHub, VirusTotal, Hunter.io, Chaos (chaos.projectdiscovery.io), WPScan, BeVigil, LeakIX, Fullhunt.

**5. Verify BBOT is installed**

```bash
bbot --version
# Should output: bbot 2.8.x
```

If not installed:

```bash
pip install bbot
# or
pipx install bbot
```

## Usage

Once installed, activate by saying things like:

- "Run BBOT scan for [company]"
- "Enumerate subdomains for [target]"
- "Check for subdomain takeover on [target]"
- "Scan for S3 buckets for [company]"
- "Run nuclei on [target]"
- "Passive recon on [target]"
- "Full engagement on [target]"
- "Mobile app recon for [company]"
- "Decompile APK for [app]"
- "Hunt for leaked credentials for [company]"
- "Azure blob storage scan for [target]"
