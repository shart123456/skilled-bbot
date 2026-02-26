# claude-bbot-skill

A Claude Code skill for running BBOT (Bighuge BLS OSINT Tool) scans during bug bounty reconnaissance.

## What it does

Provides Claude with full context to execute intelligent BBOT scans that stay within program scope, respect rate limits, and select appropriate modules based on the engagement type (passive, safe-active, or thorough).

Covers all 117+ BBOT modules organized by category:

- DNS enumeration
- Port scanning
- Web scanning and fuzzing
- Vulnerability scanning
- Cloud storage enumeration
- Code repository discovery
- Email and credential discovery
- Output modules

## Structure

```
SKILL.md                  - Skill definition and activation triggers
CLAUDE.md                 - Full BBOT technical reference
modules/                  - Module documentation by category
workflows/                - Pre-built scan workflows for common scenarios
```

## Workflows

| Workflow | Description |
|---|---|
| `passive-recon` | No direct target interaction, API sources only |
| `safe-active-recon` | Light active scanning, safe for most programs |
| `full-engagement` | Comprehensive scan for permissive scopes |
| `subdomain-takeover` | Detect dangling DNS and takeover candidates |
| `vuln-scan` | Nuclei + badsecrets + baddns on live hosts |
| `cloud-hunt` | S3, GCS, Azure blob enumeration |
| `web-app-audit` | Parameter mining, secret detection, web fuzzing |
| `code-leak-hunt` | GitHub, GitLab, Postman, Trufflehog |

## Requirements

- BBOT v2.8.2+ installed at `/opt/bbot/`
- Claude Code with skills support

## Setup After Cloning

**1. Clone the repo**

```bash
git clone git@github.com:shart123456/claude-bbot-skill.git
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

The passive recon workflow benefits from several API keys. Add these to your shell profile (`~/.bashrc` or `~/.zshrc`):

```bash
export GITHUB_TOKEN="ghp_..."
export SHODAN_KEY="..."
export VT_KEY="..."                # VirusTotal
export ST_KEY="..."                # SecurityTrails
export CHAOS_KEY="..."             # ProjectDiscovery Chaos
export HUNTER_KEY="..."            # Hunter.io
export CENSYS_ID="..."
export CENSYS_SECRET="..."
export POSTMAN_KEY="..."
export WPSCAN_KEY="..."
```

Free API keys available at: GitHub, VirusTotal, Hunter.io, Chaos (chaos.projectdiscovery.io), WPScan.

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
