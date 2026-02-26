# BBot Web Scanning & Fuzzing Modules

Complete reference for all web scanning, fuzzing, parameter mining, and web discovery modules.

---

## httpx

**Description:** The core web module. Visits webpages via HTTP/HTTPS probing, follows redirects, and generates URL and HTTP_RESPONSE events. Most other web modules depend on httpx running first. Uses the ProjectDiscovery httpx binary.

**Flags:** `active`, `safe`, `web-basic`, `social-enum`, `subdomain-enum`, `cloud-enum`
**Watched Events:** `OPEN_TCP_PORT`, `URL_UNVERIFIED`, `URL`
**Produced Events:** `URL`, `HTTP_RESPONSE`

**Dependencies:** `httpx` binary v1.2.5+ (auto-downloaded from GitHub releases)

**Options:**
| Option | Default | Description |
|---|---|---|
| `threads` | `50` | Number of concurrent HTTP probes |
| `in_scope_only` | `True` | Only probe in-scope URLs |
| `version` | `"1.2.5"` | httpx binary version to use |
| `max_response_size` | `5242880` | Max HTTP response size in bytes (5MB) |
| `store_responses` | `False` | Save full HTTP responses to disk |
| `probe_all_ips` | `False` | Probe all IPs for a hostname (not just first) |

**Usage Examples:**
```bash
# Basic web probing
bbot -t example.com -m httpx

# Increase threads for speed
bbot -t example.com -m httpx -c modules.httpx.threads=100

# Store all responses to disk
bbot -t example.com -m httpx -c modules.httpx.store_responses=true

# Probe every IP for multi-homed hosts
bbot -t example.com -m httpx -c modules.httpx.probe_all_ips=true

# With port scanning (discovers ports then probes)
bbot -t example.com -m portscan httpx sslcert
```

**HTTP_RESPONSE Event Data:**
```json
{
  "url": "https://example.com",
  "status_code": 200,
  "content_type": "text/html",
  "title": "Welcome to Example",
  "server": "nginx/1.18.0",
  "headers": {...},
  "body_hash": "sha256:...",
  "technologies": ["nginx", "Bootstrap"]
}
```

---

## ffuf

**Description:** Fast web fuzzer written in Go. Discovers hidden directories, files, and endpoints on web servers by brute-forcing paths from a wordlist.

**Flags:** `aggressive`, `active`, `deadly`
**Watched Events:** `URL`
**Produced Events:** `URL_UNVERIFIED`

**Dependencies:** `ffuf` binary v2.1.0+ (system or auto-installed)

**Options:**
| Option | Default | Description |
|---|---|---|
| `wordlist` | SecLists raft-small-directories.txt (URL) | Path or URL to fuzzing wordlist |
| `lines` | `5000` | Max number of lines to use from wordlist |
| `max_depth` | `0` | Recursion depth (0 = no recursion) |
| `extensions` | `""` | File extensions to fuzz (e.g., `"php,asp,html"`) |
| `ignore_case` | `False` | Case-insensitive matching |
| `rate` | `0` | Max requests/second (0 = unlimited) |

**Usage Examples:**
```bash
# Basic directory fuzzing
bbot -t example.com -m httpx ffuf

# With extensions
bbot -t example.com -m httpx ffuf \
     -c modules.ffuf.extensions="php,asp,txt,bak"

# Rate-limited for respectful scanning
bbot -t example.com -m httpx ffuf \
     -c modules.ffuf.rate=50 \
        modules.ffuf.lines=2000

# Deep recursion
bbot -t example.com -m httpx ffuf \
     -c modules.ffuf.max_depth=3 \
        modules.ffuf.wordlist=/opt/wordlists/raft-large-directories.txt

# Custom wordlist
bbot -t example.com -m httpx ffuf \
     -c modules.ffuf.wordlist=/opt/SecLists/Discovery/Web-Content/big.txt \
        modules.ffuf.lines=20000
```

**Warning:** ffuf is flagged `deadly` — only use on targets where web fuzzing is explicitly permitted.

---

## ffuf_shortnames

**Description:** Specialized ffuf-based module for exploiting IIS shortname (8.3 filename) vulnerabilities. Guesses full filenames from discovered shortnames and finds hidden files on IIS servers.

**Flags:** `aggressive`, `active`, `iis-shortnames`
**Watched Events:** `URL`
**Produced Events:** `URL_UNVERIFIED`

**Dependencies:** `ffuf` binary

**Options:**
| Option | Default | Description |
|---|---|---|
| `wordlist` | SecLists IIS shortnames wordlist (URL) | Wordlist specific to IIS shortname guessing |
| `lines` | `5000` | Lines from wordlist to use |

**Works With:** `iis_shortnames` module — first detect shortnames, then brute-force with `ffuf_shortnames`

```bash
# Detect IIS shortnames then exploit
bbot -t example.com -m httpx iis_shortnames ffuf_shortnames
```

---

## iis_shortnames

**Description:** Detects IIS 8.3 shortname disclosure vulnerability. IIS servers may reveal truncated (8.3 format) filenames of files and directories via HTTP 404 responses, disclosing hidden content.

**Flags:** `active`, `safe`, `iis-shortnames`
**Watched Events:** `URL`
**Produced Events:** `FINDING`

**Options:**
| Option | Default | Description |
|---|---|---|
| `detect_only` | `True` | Only detect, don't enumerate shortnames |

```bash
bbot -t example.com -m httpx iis_shortnames

# Full detection + enumeration + guessing
bbot -t example.com -m httpx iis_shortnames ffuf_shortnames
```

---

## bypass403

**Description:** Tests various techniques to bypass 403 Forbidden responses. Attempts path normalization, header injection, method manipulation, and IP spoofing techniques to access restricted resources.

**Flags:** `active`, `aggressive`
**Watched Events:** `URL`
**Produced Events:** `FINDING`

**Bypass Techniques Tested:**
- Path normalization: `//path`, `/./path`, `/.//path`
- URL encoding: `/%2Fpath`, `/%252Fpath`
- Header injection: `X-Forwarded-For: 127.0.0.1`, `X-Real-IP: 127.0.0.1`
- HTTP method changes: `HEAD`, `OPTIONS`, `TRACE`
- Rewrite headers: `X-Rewrite-URL`, `X-Original-URL`

```bash
bbot -t example.com -m httpx bypass403
```

---

## vhost

**Description:** Virtual host fuzzing — discovers hidden virtual hosts by sending requests with different `Host` headers and looking for different responses. Reveals internal applications served on the same IP.

**Flags:** `active`, `aggressive`, `subdomain-enum`
**Watched Events:** `IP_ADDRESS`, `DNS_NAME`
**Produced Events:** `DNS_NAME`

**Options:**
| Option | Default | Description |
|---|---|---|
| `wordlist` | SecLists subdomains wordlist (URL) | Wordlist of potential virtual hostnames |
| `lines` | `5000` | Lines to use from wordlist |
| `concurrency` | `10` | Concurrent vhost probes |

```bash
bbot -t example.com -m httpx vhost \
     -c modules.vhost.wordlist=/opt/SecLists/Discovery/DNS/subdomains-top1million-5000.txt
```

---

## host_header

**Description:** Tests for Host header injection vulnerabilities. Sends requests with manipulated Host headers and monitors responses for signs that the application uses the Host header in server-side operations (password reset links, redirects, etc.).

**Flags:** `active`, `aggressive`
**Watched Events:** `URL`
**Produced Events:** `FINDING`

```bash
bbot -t example.com -m httpx host_header
```

---

## robots

**Description:** Fetches and parses `robots.txt` files. Extracts `Disallow` and `Allow` entries as URL paths, often revealing hidden directories, admin panels, API endpoints, and sensitive paths.

**Flags:** `active`, `safe`, `web-basic`
**Watched Events:** `URL`
**Produced Events:** `URL_UNVERIFIED`

```bash
bbot -t example.com -m httpx robots
```

**What to Expect:**
- Admin panels: `/admin/`, `/wp-admin/`, `/backend/`
- API endpoints: `/api/v1/`, `/graphql`, `/swagger`
- Sensitive directories: `/backup/`, `/staging/`, `/.git/`
- Hidden files: `/config.php`, `/.env`

---

## securitytxt

**Description:** Fetches and parses `security.txt` files (RFC 9116). Extracts contact information, PGP keys, security policy URLs, and sometimes bug bounty platform links.

**Flags:** `active`, `safe`, `web-basic`
**Watched Events:** `URL`
**Produced Events:** `EMAIL_ADDRESS`, `URL_UNVERIFIED`, `FINDING`

```bash
bbot -t example.com -m httpx securitytxt
```

**Parsed Fields:**
- `Contact:` — Security contact email/URL
- `Encryption:` — PGP key URL
- `Policy:` — Vulnerability disclosure policy URL
- `Bug-Bounty:` — Bug bounty program URL
- `Expires:` — Policy expiration
- `Acknowledgments:` — Hall of fame URL

---

## url_manipulation

**Description:** Tests URL manipulation vulnerabilities. Attempts path traversal, parameter tampering, and URL rewriting issues that may expose unexpected behavior or unauthorized access.

**Flags:** `active`, `aggressive`
**Watched Events:** `URL`
**Produced Events:** `FINDING`

```bash
bbot -t example.com -m httpx url_manipulation
```

---

## paramminer_getparams

**Description:** Brute-forces hidden GET parameters. Sends requests with many GET parameters at once using binary search to efficiently discover parameters the server responds differently to. Essential for finding hidden API parameters and debug flags.

**Flags:** `active`, `aggressive`, `web-paramminer`
**Watched Events:** `HTTP_RESPONSE`
**Produced Events:** `WEB_PARAMETER`

**Options:**
| Option | Default | Description |
|---|---|---|
| `wordlist` | SecLists burp-parameter-names.txt | Parameter name wordlist |
| `recycle_words` | `True` | Reuse words from response content as potential parameters |
| `ignore_params` | `[]` | Parameters to ignore |
| `http_extract_params` | `True` | Extract parameters from HTTP responses |

```bash
bbot -t example.com -m httpx paramminer_getparams

# With custom wordlist
bbot -t example.com -m httpx paramminer_getparams \
     -c modules.paramminer_getparams.wordlist=/opt/wordlists/params.txt
```

---

## paramminer_headers

**Description:** Brute-forces hidden HTTP request headers. Discovers non-standard headers that alter server behavior — often revealing debug modes, internal routing, feature flags, and header-based injection points.

**Flags:** `active`, `aggressive`, `web-paramminer`
**Watched Events:** `HTTP_RESPONSE`
**Produced Events:** `WEB_PARAMETER`

**Options:**
| Option | Default | Description |
|---|---|---|
| `wordlist` | SecLists http-request-headers.txt | Header name wordlist |
| `recycle_words` | `True` | Extract potential header names from responses |

```bash
bbot -t example.com -m httpx paramminer_headers
```

**Interesting Headers Often Found:**
- `X-Debug: true` — Enable debug output
- `X-Forward-For: 127.0.0.1` — Bypass IP restrictions
- `X-Custom-IP-Authorization` — Auth bypass
- `X-Admin: true` — Admin access
- `X-API-Version: 2` — Alternative API versions

---

## paramminer_cookies

**Description:** Brute-forces hidden cookie names. Discovers undocumented cookies that alter application behavior — often revealing feature flags, debug modes, user roles, and session variations.

**Flags:** `active`, `aggressive`, `web-paramminer`
**Watched Events:** `HTTP_RESPONSE`
**Produced Events:** `WEB_PARAMETER`

**Options:**
| Option | Default | Description |
|---|---|---|
| `wordlist` | SecLists cookie-names.txt | Cookie name wordlist |
| `recycle_words` | `True` | Use response-derived words as cookie names |

```bash
bbot -t example.com -m httpx paramminer_cookies
```

---

## reflected_parameters

**Description:** Tests discovered web parameters for reflection in HTTP responses. Parameters that reflect user input are candidates for XSS, template injection, and other injection vulnerabilities.

**Flags:** `active`, `aggressive`
**Watched Events:** `WEB_PARAMETER`
**Produced Events:** `FINDING`

```bash
# Full pipeline: discover params, then test reflection
bbot -t example.com -m httpx paramminer_getparams reflected_parameters
```

---

## smuggler

**Description:** Tests for HTTP request smuggling vulnerabilities. Attempts CL.TE and TE.CL smuggling attacks by sending ambiguous request boundaries and detecting differential responses.

**Flags:** `active`, `aggressive`
**Watched Events:** `URL`
**Produced Events:** `FINDING`, `VULNERABILITY`

```bash
bbot -t example.com -m httpx smuggler
```

**Smuggling Types Tested:**
- CL.TE (Content-Length + Transfer-Encoding)
- TE.CL (Transfer-Encoding + Content-Length)
- TE.TE (obfuscated Transfer-Encoding)
- HTTP/2 downgrade smuggling

---

## gowitness

**Description:** Takes screenshots of web pages using a headless Chromium browser. Produces WEBSCREENSHOT events with base64-encoded images. Essential for visual survey of large attack surfaces.

**Flags:** `active`, `safe`, `web-screenshots`
**Watched Events:** `URL`, `SOCIAL`
**Produced Events:** `WEBSCREENSHOT`, `URL`, `URL_UNVERIFIED`, `TECHNOLOGY`

**Dependencies:** `chromium` (system), `aiosqlite` (Python), `gowitness` binary v3.0.5+

**Options:**
| Option | Default | Description |
|---|---|---|
| `version` | `"3.0.5"` | gowitness binary version |
| `threads` | `0` | Concurrent screenshot threads (0 = auto) |
| `timeout` | `10` | Page load timeout in seconds |
| `resolution_x` | `1440` | Screenshot width in pixels |
| `resolution_y` | `900` | Screenshot height in pixels |
| `output_path` | `""` | Directory to save screenshots (auto-created) |
| `social` | `False` | Also screenshot social media profile URLs |
| `idle_timeout` | `1800` | Shutdown if idle for this many seconds |
| `chrome_path` | `""` | Custom Chromium binary path |

**Usage Examples:**
```bash
# Screenshot all discovered URLs
bbot -t example.com -m httpx gowitness

# High-resolution screenshots
bbot -t example.com -m httpx gowitness \
     -c modules.gowitness.resolution_x=1920 \
        modules.gowitness.resolution_y=1080

# With social profile screenshots
bbot -t example.com -m httpx social gowitness \
     -c modules.gowitness.social=true

# Full visual survey
bbot -t example.com \
     -f subdomain-enum web-basic web-screenshots \
     -n visual_survey
```

---

## social

**Description:** Finds social media profile links embedded in web pages. Identifies LinkedIn, Twitter/X, GitHub, Facebook, Instagram, YouTube channels, and other social media references in HTML content.

**Flags:** `active`, `safe`, `social-enum`
**Watched Events:** `HTTP_RESPONSE`
**Produced Events:** `SOCIAL`

**Social Platforms Detected:**
- LinkedIn (company profiles, personal profiles)
- Twitter/X
- GitHub (organizations, users)
- Facebook
- Instagram
- YouTube
- Discord invite links
- Slack workspaces
- Telegram
- Reddit

```bash
bbot -t example.com -m httpx social

# Then screenshot social profiles
bbot -t example.com -m httpx social gowitness \
     -c modules.gowitness.social=true
```

---

## oauth

**Description:** Enumerates OAuth 2.0 and OpenID Connect endpoints. Discovers authorization servers, token endpoints, JWKS endpoints, and OpenID Connect Discovery documents (`.well-known/openid-configuration`).

**Flags:** `active`, `safe`, `web-basic`
**Watched Events:** `URL`, `HTTP_RESPONSE`
**Produced Events:** `FINDING`, `URL_UNVERIFIED`

```bash
bbot -t example.com -m httpx oauth
```

**Endpoints Checked:**
- `/.well-known/openid-configuration`
- `/.well-known/oauth-authorization-server`
- `/oauth/authorize`, `/oauth/token`
- `/auth/oauth`, `/api/auth`
- JWKS endpoints from discovery docs

---

## newsletters

**Description:** Detects newsletter subscription forms and email submission endpoints. Identifies marketing automation integrations (Mailchimp, HubSpot, etc.) and email collection endpoints.

**Flags:** `active`, `safe`, `web-basic`
**Watched Events:** `HTTP_RESPONSE`
**Produced Events:** `FINDING`

```bash
bbot -t example.com -m httpx newsletters
```

---

## graphql_introspection

**Description:** Tests GraphQL endpoints for introspection being enabled. When introspection is enabled, it retrieves the complete GraphQL schema — all types, queries, mutations, and subscriptions — which is critical for mapping the API attack surface.

**Flags:** `active`, `aggressive`
**Watched Events:** `URL`
**Produced Events:** `FINDING`, `URL_UNVERIFIED`

**Options:**
| Option | Default | Description |
|---|---|---|
| `enabled` | `True` | Enable graphql introspection testing |

```bash
bbot -t example.com -m httpx graphql_introspection
```

**GraphQL Endpoints Tested:**
- `/graphql`
- `/api/graphql`
- `/graphql/v1`
- `/v1/graphql`
- `/query`
- `/gql`

---

## generic_ssrf

**Description:** Tests discovered URL parameters for Server-Side Request Forgery (SSRF) vulnerabilities. Injects BBOT's collaborator/callback URL into parameters and checks for DNS/HTTP callbacks.

**Flags:** `active`, `aggressive`
**Watched Events:** `WEB_PARAMETER`
**Produced Events:** `VULNERABILITY`

```bash
bbot -t example.com -m httpx paramminer_getparams generic_ssrf
```

---

## code_repository

**Description:** Searches HTTP response bodies and headers for links to code repositories (GitHub, GitLab, Bitbucket). Finding public repos linked from production sites often leads to source code leaks.

**Flags:** `active`, `safe`, `code-enum`
**Watched Events:** `HTTP_RESPONSE`
**Produced Events:** `CODE_REPOSITORY`

```bash
bbot -t example.com -m httpx code_repository
```

---

## Web Scanning Workflow Combinations

### Basic Safe Web Scan
```bash
bbot -t example.com \
     -m httpx robots securitytxt social oauth \
     -n basic_web
```

### Thorough Web Assessment
```bash
bbot -t example.com \
     -f web-thorough \
     -ef deadly \
     -n thorough_web
```

### Full Web + Parameter Mining
```bash
bbot -t example.com \
     -m httpx robots securitytxt oauth graphql_introspection \
        paramminer_getparams paramminer_headers paramminer_cookies \
        reflected_parameters \
     -c modules.httpx.store_responses=true \
     -n web_paramminer
```

### Visual Web Survey
```bash
bbot -t example.com \
     -m httpx gowitness social \
     -c modules.gowitness.resolution_x=1920 \
        modules.gowitness.resolution_y=1080 \
     -n visual_web_survey
```

### IIS Server Full Test
```bash
bbot -t example.com \
     -m httpx iis_shortnames ffuf_shortnames bypass403 \
     -n iis_test
```
