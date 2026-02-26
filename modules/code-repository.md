# BBot Code Repository Discovery Modules

Complete reference for source code discovery, repository enumeration, and mobile application modules.

---

## github_org

**Description:** Queries GitHub's API for repositories belonging to an organization and its members. Discovers public repositories that may contain source code, configuration files, secrets, and internal tooling for the target.

**Flags:** `passive`, `safe`, `subdomain-enum`, `code-enum`
**Watched Events:** `ORG_STUB`, `SOCIAL`
**Produced Events:** `CODE_REPOSITORY`

**Options:**
| Option | Default | Description |
|---|---|---|
| `api_key` | `""` | GitHub Personal Access Token (strongly recommended — without it you're rate-limited to 60 req/hr) |
| `include_members` | `True` | Also enumerate repositories from org members |
| `include_member_repos` | `False` | Include personal (non-org) repos from members |

**Usage Examples:**
```bash
# Basic org discovery
bbot -t github.com/acmecorp -m github_org \
     -c modules.github_org.api_key=ghp_TOKEN

# Include member repos
bbot -t github.com/acmecorp -m github_org \
     -c modules.github_org.api_key=ghp_TOKEN \
        modules.github_org.include_member_repos=true

# Full org enumeration triggered from domain
bbot -t acmecorp.com -m github_org github_codesearch \
     -c modules.github_org.api_key=ghp_TOKEN \
        modules.github_codesearch.api_key=ghp_TOKEN
```

**What to Look For:**
- Infrastructure-as-code repos (Terraform, Ansible, CloudFormation)
- CI/CD configuration (`.github/workflows/`, `.gitlab-ci.yml`)
- Internal tooling with hardcoded credentials
- Commit history with deleted secrets
- `config.yml` / `.env.example` files
- Docker files with credentials
- API keys in test files

---

## github_codesearch

**Description:** Searches GitHub code for instances of the target domain name. Finds repositories — often public forks, personal projects, or contractor repos — that reference the target domain in code, comments, or configuration.

**Flags:** `passive`, `safe`, `subdomain-enum`, `code-enum`
**Watched Events:** `DNS_NAME`
**Produced Events:** `CODE_REPOSITORY`, `URL_UNVERIFIED`

**Options:**
| Option | Default | Description |
|---|---|---|
| `api_key` | `""` | GitHub API key (required — code search is authenticated-only) |
| `limit` | `100` | Max results to return |

**Usage Examples:**
```bash
bbot -t example.com -m github_codesearch \
     -c modules.github_codesearch.api_key=ghp_TOKEN

# Get more results
bbot -t example.com -m github_codesearch \
     -c modules.github_codesearch.api_key=ghp_TOKEN \
        modules.github_codesearch.limit=500
```

**Search Strategies:**
- `"example.com"` in code
- `"api.example.com"` in URLs
- Config files referencing target environment
- Internal domain names (`.internal`, `.corp`)

---

## github_usersearch

**Description:** Searches GitHub for user accounts associated with company email addresses or domains. Discovers employee GitHub accounts that may have organization repos, forks, or code snippets relevant to the target.

**Flags:** `passive`, `safe`, `code-enum`
**Watched Events:** `EMAIL_ADDRESS`
**Produced Events:** `SOCIAL`

**Options:**
| Option | Default | Description |
|---|---|---|
| `api_key` | `""` | GitHub API key |

```bash
bbot -t example.com \
     -m hunterio github_usersearch \
     -c modules.hunterio.api_key=KEY \
        modules.github_usersearch.api_key=ghp_TOKEN
```

---

## github_workflows

**Description:** Discovers GitHub Actions workflow files in target repositories. Workflow files often contain deployment secrets, cloud credentials, and infrastructure configuration in environment variables.

**Flags:** `passive`, `safe`, `code-enum`
**Watched Events:** `CODE_REPOSITORY`
**Produced Events:** `FINDING`

**Options:**
| Option | Default | Description |
|---|---|---|
| `api_key` | `""` | GitHub API key |

```bash
bbot -t example.com -m github_org github_workflows \
     -c modules.github_org.api_key=ghp_TOKEN \
        modules.github_workflows.api_key=ghp_TOKEN
```

**What Workflows Reveal:**
- `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` environment variables
- Database connection strings in test workflows
- Docker registry credentials
- Internal service URLs and hostnames
- CI/CD infrastructure details

---

## gitlab_com

**Description:** Queries GitLab.com for repositories associated with the target domain or organization. Many companies use GitLab.com (public SaaS) alongside or instead of GitHub.

**Flags:** `passive`, `safe`, `code-enum`
**Watched Events:** `ORG_STUB`, `SOCIAL`
**Produced Events:** `CODE_REPOSITORY`

**Options:**
| Option | Default | Description |
|---|---|---|
| `api_key` | `""` | GitLab API token |

```bash
bbot -t example.com -m gitlab_com \
     -c modules.gitlab_com.api_key=glpat_TOKEN
```

---

## gitlab_onprem

**Description:** Detects self-hosted (on-premises) GitLab instances. Many organizations run internal GitLab servers that are accidentally internet-facing or accessible from discovered subdomains.

**Flags:** `active`, `safe`, `code-enum`
**Watched Events:** `URL`
**Produced Events:** `CODE_REPOSITORY`, `FINDING`

**Options:**
| Option | Default | Description |
|---|---|---|
| `api_key` | `""` | GitLab API token (if you have credentials) |
| `allow_unauthenticated` | `True` | Check for public repos without auth |

**Detection Method:**
- Looks for GitLab headers and page fingerprints
- Checks `/-/health` endpoint
- Checks `/api/v4/` for GitLab API

```bash
bbot -t example.com -m httpx gitlab_onprem
```

---

## postman

**Description:** Searches Postman's public API for workspaces and collections containing the target domain. Developers often share internal API documentation publicly on Postman with hardcoded credentials and internal endpoints.

**Flags:** `passive`, `safe`, `code-enum`
**Watched Events:** `DNS_NAME`
**Produced Events:** `CODE_REPOSITORY`

**Options:**
| Option | Default | Description |
|---|---|---|
| `api_key` | `""` | Postman API key (recommended for higher rate limits) |

```bash
bbot -t example.com -m postman \
     -c modules.postman.api_key=PMAK_TOKEN
```

**What Postman Collections Reveal:**
- Complete internal API documentation
- Authentication tokens in examples
- API keys hardcoded in headers
- Internal hostnames and IP addresses
- Internal API endpoint paths
- Environment variable values

---

## postman_download

**Description:** Downloads discovered Postman collections and environments. Extracts API keys, auth tokens, URLs, and other sensitive data from the downloaded collection files.

**Flags:** `active`, `safe`, `code-enum`
**Watched Events:** `CODE_REPOSITORY`
**Produced Events:** `FINDING`, `URL_UNVERIFIED`

```bash
bbot -t example.com -m postman postman_download \
     -c modules.postman.api_key=PMAK_TOKEN
```

---

## git

**Description:** Detects exposed `.git` directories on web servers. When a web server accidentally serves the `.git/` directory, attackers can reconstruct the entire source code.

**Flags:** `active`, `safe`, `web-basic`
**Watched Events:** `URL`
**Produced Events:** `FINDING`

**Detection Method:**
- Checks for `/.git/config` (returns 200 with git config)
- Checks for `/.git/HEAD` file
- Checks for `/.git/COMMIT_EDITMSG`

```bash
bbot -t example.com -m httpx git
```

---

## git_clone

**Description:** Clones discovered git repositories to local disk for analysis. Works with both `github_org` discovered repos and `git` (exposed .git directory) findings.

**Flags:** `active`, `safe`, `code-enum`
**Watched Events:** `CODE_REPOSITORY`
**Produced Events:** (saves to disk)

**Options:**
| Option | Default | Description |
|---|---|---|
| `output_folder` | `$BBOT_HOME/repos/` | Where to save cloned repositories |
| `api_key` | `""` | GitHub/GitLab token for private repos |

```bash
bbot -t example.com -m github_org git_clone \
     -c modules.github_org.api_key=ghp_TOKEN \
        modules.git_clone.api_key=ghp_TOKEN \
        modules.git_clone.output_folder=/opt/repos/example/
```

---

## gitdumper

**Description:** Dumps exposed `.git` directories from web servers. Uses the `git-dumper` technique to reconstruct source code from a partially exposed `.git/` directory even when directory listing is disabled.

**Flags:** `active`, `aggressive`
**Watched Events:** `FINDING` (from `git` module)
**Produced Events:** `FINDING`

**How It Works:**
1. Fetches `/.git/config` to verify existence
2. Downloads `/.git/COMMIT_EDITMSG` to get latest commit hash
3. Fetches `/.git/objects/` tree recursively
4. Reconstructs working directory from object store

```bash
bbot -t example.com -m httpx git gitdumper
```

---

## dockerhub

**Description:** Searches DockerHub for container images associated with the target organization. Docker images often contain application source code, configuration files, hardcoded secrets, and internal infrastructure details.

**Flags:** `passive`, `safe`, `code-enum`
**Watched Events:** `ORG_STUB`, `SOCIAL`
**Produced Events:** `CODE_REPOSITORY`

**Options:**
| Option | Default | Description |
|---|---|---|
| `limit` | `10` | Max results per query |

```bash
bbot -t example.com -m dockerhub
```

---

## docker_pull

**Description:** Pulls Docker images discovered by `dockerhub` and analyzes their contents. Extracts environment variables, file system contents, and embedded secrets from Docker image layers.

**Flags:** `active`, `aggressive`
**Watched Events:** `CODE_REPOSITORY`
**Produced Events:** `FINDING`

```bash
bbot -t example.com -m dockerhub docker_pull
```

---

## google_playstore

**Description:** Searches Google Play Store for mobile applications associated with the target organization. Mobile apps often contain hardcoded API endpoints, internal subdomains, and credentials.

**Flags:** `passive`, `safe`, `code-enum`
**Watched Events:** `ORG_STUB`, `DNS_NAME`
**Produced Events:** `CODE_REPOSITORY`

```bash
bbot -t example.com -m google_playstore
```

---

## apkpure

**Description:** Downloads Android APK files from APKPure.com for analysis. APKPure hosts APKs that may not be available on the official Play Store.

**Flags:** `active`, `safe`, `code-enum`
**Watched Events:** `CODE_REPOSITORY`
**Produced Events:** `FINDING`

```bash
bbot -t example.com -m google_playstore apkpure jadx
```

---

## jadx

**Description:** Decompiles Android APK files using JADX. Converts compiled Dalvik bytecode back to readable Java/Kotlin source code for analysis of hardcoded secrets, API endpoints, and application logic.

**Flags:** `active`, `safe`, `code-enum`
**Watched Events:** `CODE_REPOSITORY`
**Produced Events:** `FINDING`

**Dependencies:** `jadx` (system tool)

**What JADX Finds:**
- Hardcoded API keys and secrets
- Internal API endpoints and base URLs
- AWS/GCP/Azure credentials
- Database connection strings
- Internal domain names
- Cryptographic keys

```bash
bbot -t example.com -m google_playstore jadx
```

---

## Code Repository Workflows

### Full GitHub Intelligence Gathering
```bash
bbot -t example.com \
     -m github_org github_codesearch github_workflows \
        github_usersearch git_clone \
     -c modules.github_org.api_key=ghp_TOKEN \
        modules.github_org.include_member_repos=true \
        modules.github_codesearch.api_key=ghp_TOKEN \
        modules.github_workflows.api_key=ghp_TOKEN \
        modules.github_usersearch.api_key=ghp_TOKEN \
        modules.git_clone.api_key=ghp_TOKEN \
     -n github_full
```

### Exposed Git Directory Hunt
```bash
bbot -t example.com \
     -m httpx git gitdumper \
     -n git_exposure_hunt
```

### Postman Intelligence
```bash
bbot -t example.com \
     -m postman postman_download \
     -c modules.postman.api_key=PMAK_TOKEN \
     -n postman_intel
```

### Full Code Preset
```bash
bbot -t example.com -p code-enum \
     -c modules.github_org.api_key=ghp_TOKEN \
        modules.github_codesearch.api_key=ghp_TOKEN
```

### Mobile App Analysis
```bash
bbot -t example.com \
     -m google_playstore apkpure jadx \
     -n mobile_app_analysis
```

### Full Code Leak Hunt
```bash
bbot -t example.com \
     -m github_org github_codesearch gitlab_com postman postman_download \
        dockerhub docker_pull google_playstore \
        httpx git gitdumper code_repository \
     -c modules.github_org.api_key=ghp_TOKEN \
        modules.github_codesearch.api_key=ghp_TOKEN \
        modules.gitlab_com.api_key=glpat_TOKEN \
        modules.postman.api_key=PMAK_TOKEN \
     -n full_code_leak_hunt
```
