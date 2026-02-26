---
name: IP Intelligence and Technology Detection Modules
description: Complete reference for IP geolocation, WAF detection, technology fingerprinting, and utility modules
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
# BBot IP Intelligence & Technology Detection Modules

Complete reference for IP geolocation, WAF detection, technology fingerprinting, and utility modules.

---

## asn

**Description:** Performs ASN (Autonomous System Number) lookups for discovered IP addresses and expands ASN ranges into IP CIDR blocks. Useful for mapping the full IP space owned by a target organization.

**Flags:** `passive`, `safe`
**Watched Events:** `IP_ADDRESS`
**Produced Events:** `ASN`, `IP_RANGE`

**No API key required. No configuration options.**

```bash
bbot -t example.com -m asn
```

**Data Returned:**
- ASN number and organization name
- Associated CIDR ranges (IP_RANGE events)
- Country and registry information

> **Scope Warning:** The `asn` module can **massively expand your target surface**. If `example.com` resolves to an IP owned by AWS or Cloudflare, `asn` will produce IP_RANGE events covering millions of IPs. Always combine with `--blacklist` and careful scope review. The `speculate` internal module will then generate targets from these ranges.
>
> Best practice: use `asn` only when you have confirmed CIDR ownership via `whois` or program scope documentation.

```bash
# Combined with scope controls
bbot -t 1.2.3.4 -m asn \
     --blacklist 1.2.0.0/16 \
     -n asn_lookup
```

---

## cloudcheck

**Description:** Tags discovered IP addresses with their cloud provider. Identifies whether an IP belongs to AWS, GCP, Azure, Cloudflare, Fastly, or other major cloud/CDN providers by cross-referencing published cloud IP ranges.

**Flags:** `passive`, `safe`
**Watched Events:** `IP_ADDRESS`
**Produced Events:** `FINDING` (cloud provider tag)

**No API key required. No configuration options.**

```bash
# cloudcheck runs automatically during most scans
# Explicit invocation:
bbot -t example.com -m cloudcheck
```

**Why it matters:**
- Suppresses CDN false positives in port scan results — ports open on Cloudflare IPs aren't the target's attack surface
- Identifies direct-to-origin IPs vs CDN-fronted IPs
- Guides which IPs to port-scan vs skip
- Works alongside `portscan` — cloudcheck events help filter portscan scope

**Cloud Providers Detected:**
- AWS (EC2, S3, CloudFront)
- Google Cloud (GCE, Firebase, GKE)
- Microsoft Azure
- Cloudflare
- Fastly
- Akamai
- DigitalOcean
- Oracle Cloud

---

## ip2location

**Description:** Queries ip2location.io for geolocation data for discovered IP addresses. Returns city, region, country, ISP, coordinates, and autonomous system information.

**Flags:** `passive`, `safe`
**Watched Events:** `IP_ADDRESS`
**Produced Events:** `GEOLOCATION`

**Options:**
| Option | Default | Description |
|---|---|---|
| `api_key` | `""` | ip2location.io API key (free tier: 30,000 queries/month) |

**Data Returned:**
- City, region, country
- Latitude/longitude coordinates
- ISP and organization name
- Autonomous System Number (ASN)
- Domain associated with IP
- Usage type (Data Center, Commercial, Residential, etc.)
- Threat intelligence tags (proxy, VPN, TOR, etc.)
- Time zone

```bash
bbot -t example.com -m ip2location \
     -c modules.ip2location.api_key=KEY
```

---

## ipstack

**Description:** Queries ipstack.com for IP address geolocation and intelligence data. Alternative to ip2location with different coverage and data sources.

**Flags:** `passive`, `safe`
**Watched Events:** `IP_ADDRESS`
**Produced Events:** `GEOLOCATION`

**Options:**
| Option | Default | Description |
|---|---|---|
| `api_key` | `""` | ipstack API key (free tier: 100 requests/month) |

**Data Returned:**
- IP address type (IPv4/IPv6)
- Country, region, city
- Zip code
- Latitude/longitude
- Time zone
- Connection type
- ISP name

```bash
bbot -t example.com -m ipstack \
     -c modules.ipstack.api_key=KEY
```

---

## digitorus

**Description:** Queries the Digitorus domain reputation API. Provides domain and IP reputation scoring, threat intelligence, and associated domain data.

**Flags:** `passive`, `safe`
**Watched Events:** `DNS_NAME`
**Produced Events:** `FINDING`

```bash
bbot -t example.com -m digitorus
```

---

## builtwith

**Description:** Queries BuiltWith.com's technology profiler API. BuiltWith maintains an extensive database of which web technologies are used by each website, updated via internet-wide crawling.

**Flags:** `passive`, `safe`, `code-enum`
**Watched Events:** `DNS_NAME`
**Produced Events:** `TECHNOLOGY`, `URL_UNVERIFIED`

**Options:**
| Option | Default | Description |
|---|---|---|
| `api_key` | `""` | BuiltWith API key (paid service) |

**Technology Categories Detected:**
- Web servers (Apache, nginx, IIS, Caddy)
- Frontend frameworks (React, Angular, Vue, jQuery)
- Backend frameworks (Django, Rails, Laravel, ASP.NET)
- CMS platforms (WordPress, Drupal, Joomla, Ghost)
- Analytics (Google Analytics, Mixpanel, Segment)
- CDN providers (Cloudflare, Fastly, Akamai, CloudFront)
- Hosting (AWS, Azure, GCP, Heroku)
- Payment processors (Stripe, PayPal, Braintree)
- Email services (SendGrid, Mailchimp, SES)
- Security (reCAPTCHA, Cloudflare Bot Management)
- 200+ additional technology categories

```bash
bbot -t example.com -m builtwith \
     -c modules.builtwith.api_key=KEY
```

---

## ironsight

**Description:** Technology fingerprinting using IronSight's detection engine. Identifies web technologies, frameworks, and software from HTTP response headers, HTML content, and JavaScript files.

**Flags:** `active`, `safe`, `web-basic`
**Watched Events:** `HTTP_RESPONSE`
**Produced Events:** `TECHNOLOGY`

**Detection Method:**
- HTTP response header analysis
- HTML meta tag detection
- JavaScript file fingerprinting
- Cookie name analysis
- URL pattern matching
- Error page content analysis

**Technologies Detected:**
- Web application frameworks
- CMS platforms
- Server software and versions
- Security products (WAF, CDN)
- JavaScript libraries with versions
- Database indicators

```bash
bbot -t example.com -m httpx ironsight
```

---

## bevigil

**Description:** Queries BeVigil — an OSINT search engine focused on mobile app security. Searches for APIs, endpoints, secrets, and domains discovered through mobile application analysis.

**Flags:** `passive`, `safe`, `code-enum`
**Watched Events:** `DNS_NAME`
**Produced Events:** `URL_UNVERIFIED`, `FINDING`

**Options:**
| Option | Default | Description |
|---|---|---|
| `api_key` | `""` | BeVigil API key (free tier available) |

**BeVigil Data Sources:**
- Android APK analysis
- iOS IPA analysis
- Extracted API endpoints
- Hardcoded URLs and secrets
- Mobile app permissions

```bash
bbot -t example.com -m bevigil \
     -c modules.bevigil.api_key=KEY
```

---

## wappalyzer

**Description:** Technology fingerprinting using the Wappalyzer rules database. Identifies web technologies, CMS platforms, frameworks, and libraries from HTTP responses by matching against Wappalyzer's extensive pattern library.

**Flags:** `active`, `safe`, `web-basic`
**Watched Events:** `HTTP_RESPONSE`
**Produced Events:** `TECHNOLOGY`

**No API key required. No configuration options.**

```bash
bbot -t example.com -m httpx wappalyzer
```

**Technologies Detected (same categories as Wappalyzer browser extension):**
- Web servers (Apache, nginx, IIS, Caddy, LiteSpeed)
- Frontend frameworks (React, Angular, Vue.js, jQuery, Bootstrap)
- Backend frameworks (Django, Rails, Laravel, Express, ASP.NET)
- CMS (WordPress, Drupal, Joomla, Ghost, Contentful)
- E-commerce (Shopify, WooCommerce, Magento, BigCommerce)
- JavaScript libraries with version detection
- CDN / reverse proxy detection
- Analytics and marketing tools
- Security products (reCAPTCHA, hCaptcha, Cloudflare Bot Management)

**Contrast with `ironsight`:**
- `wappalyzer` uses the Wappalyzer rules database (community-maintained, ~1500+ technologies)
- `ironsight` uses IronSight's proprietary detection engine
- Running both provides better coverage: `bbot -t example.com -m httpx ironsight wappalyzer`

---

## wafw00f

**Description:** Web Application Firewall fingerprinting. Identifies which WAF or IPS is protecting a web application by analyzing HTTP responses and behavior. Knowing the WAF helps tailor testing techniques.

**Flags:** `active`, `aggressive`
**Watched Events:** `URL`
**Produced Events:** `WAF`

**Dependencies:** `wafw00f~=2.3.1` (Python)

**Options:**
| Option | Default | Description |
|---|---|---|
| `generic_detect` | `True` | Enable generic WAF detection (even if vendor unknown) |

**WAFs Detected (80+):**
- Cloudflare
- AWS WAF (Amazon Web Services)
- Azure Application Gateway WAF
- Akamai Kona Site Defender
- Fastly
- Sucuri
- Imperva Incapsula
- Barracuda Web Application Firewall
- F5 BIG-IP ASM
- Citrix NetScaler AppFW
- Fortinet FortiWeb
- ModSecurity (Apache)
- NAXSI (nginx)
- Radware AppWall
- Palo Alto Networks
- 65+ more vendors

```bash
bbot -t example.com -m httpx wafw00f
```

**Why WAF Detection Matters:**
- Different WAFs have different bypass techniques
- Cloudflare vs AWS WAF require different evasion strategies
- Identifies if target has active security monitoring
- WAF type indicates infrastructure stack

---

## extractous

**Description:** Extracts and parses data from downloaded files (PDFs, Office documents, etc.). When used with `filedownload`, analyzes downloaded files for embedded URLs, metadata, and sensitive content.

**Flags:** `active`, `safe`
**Watched Events:** `FILESYSTEM`
**Produced Events:** `URL_UNVERIFIED`, `FINDING`

```bash
bbot -t example.com -m filedownload extractous
```

**Supported File Types:**
- PDF documents (`.pdf`)
- Microsoft Office (`.docx`, `.xlsx`, `.pptx`)
- OpenDocument Format (`.odt`, `.ods`, `.odp`)
- Plain text files (`.txt`, `.csv`, `.log`)
- Email files (`.eml`, `.msg`)

---

## filedownload

**Description:** Downloads files discovered during scanning (PDFs, documents, archives) for local analysis. Useful when combined with `extractous` to extract URLs and metadata from downloaded content.

**Flags:** `active`, `safe`
**Watched Events:** `URL`
**Produced Events:** `FILESYSTEM`

**Options:**
| Option | Default | Description |
|---|---|---|
| `extensions` | `pdf,docx,doc,xlsx,xls,pptx,ppt,txt,csv,zip,tar,gz,7z` | File extensions to download |
| `max_size` | `10485760` | Max file size to download (10MB) |
| `output_folder` | `$BBOT_HOME/downloads/` | Where to save downloaded files |

```bash
bbot -t example.com -m httpx filedownload extractous \
     -c modules.filedownload.extensions="pdf,docx,xlsx,txt" \
        modules.filedownload.output_folder=/opt/downloads/example/
```

---

## IP Intelligence Workflows

### Full IP Intelligence Profile
```bash
bbot -t example.com \
     -m ip2location shodan_idb censys_ip digitorus \
     -c modules.ip2location.api_key=KEY \
        modules.censys_ip.api_id=ID \
        modules.censys_ip.api_secret=SECRET \
     -n ip_intel
```

### Technology Stack Discovery
```bash
bbot -t example.com \
     -m httpx ironsight builtwith wafw00f bevigil \
     -c modules.builtwith.api_key=KEY \
        modules.bevigil.api_key=KEY \
     -n tech_stack_discovery
```

### Full Tech Detect Preset
```bash
bbot -t example.com -p tech-detect
```

### WAF Bypass Preparation
```bash
# Identify WAF first
bbot -t example.com -m httpx wafw00f ironsight \
     -n waf_detection

# Then run targeted tests considering WAF type
# Output includes WAF vendor in TECHNOLOGY events
```

### File Intelligence
```bash
bbot -t example.com \
     -m httpx filedownload extractous \
     -c modules.filedownload.extensions="pdf,docx,xlsx,txt,csv" \
        modules.filedownload.max_size=52428800 \
     -n file_intelligence
```
