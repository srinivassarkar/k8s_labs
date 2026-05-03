# 🌐 DNS: From First Principles to Kubernetes-Ready


> *"DNS is not magic. It's a phone book with a very specific chain of custody. Once you see the hierarchy clearly, everything else — CoreDNS, service discovery, FQDN, ndots — falls into place."*

---

## 0. First Principles

The mental model that never changes:

```
Humans use names. Machines use IPs.
DNS is the translator — globally distributed, hierarchically organized.

Resolution always walks the same path:
  Local Cache → Resolver → Root → TLD → Authoritative NS → Answer

No step invents information. Each step either answers or delegates.
```

**The hierarchy is an inverted tree, read right to left:**

```
www . kubernetes . io .
 │         │        │  └── Root (.)     — knows where TLDs live
 │         │        └───── TLD (io)     — knows where kubernetes.io lives
 │         └────────────── Domain       — owns actual DNS records
 └──────────────────────── Subdomain    — an entry inside the domain's zone
```

**Three immutable truths:**

1. **Root and TLD servers never store IPs.** They only store pointers (NS records) to the next level.
2. **Only authoritative nameservers hold real answers** (A, AAAA, CNAME, MX records).
3. **Caching is everywhere** — local OS, resolver, sometimes the browser. TTL governs freshness.

---

## 1. Reality Constraints

What DNS actually does and doesn't do:

| Claim | Reality |
|---|---|
| "DNS returns an IP immediately" | ❌ It walks up to 12 steps through the hierarchy on a cache miss |
| "Changing DNS propagates instantly" | ❌ Old TTL values must expire on all caches first |
| "Root servers store all domain IPs" | ❌ They only store NS records for TLDs — nothing else |
| "There are exactly 13 root servers" | ⚠️ 13 *named* root servers (A–M), but hundreds of physical machines via Anycast |
| "Your registrar handles DNS" | ⚠️ Only if you don't delegate to a separate DNS host. These are distinct roles. |
| "www is special" | ❌ `www` is just a subdomain — the same as `docs` or `blog`. No magic. |
| "TTL changes apply immediately" | ❌ Resolvers that already cached the old TTL will honor it until expiry |
| "Authoritative servers cache responses" | ❌ They serve directly from zone files. Only recursive resolvers cache. |
| "One zone per domain" | ❌ Large domains split into multiple zones via NS delegation of subdomains |

**Valid priority values (for reference in Kubernetes CoreDNS context):**

| Server Type | Caches? | Recurses? | Stores Records? |
|---|---|---|---|
| Recursive Resolver (ISP / 8.8.8.8 / 1.1.1.1) | ✅ | ✅ | ❌ |
| Root Nameserver | ❌ | ❌ | NS for TLDs only |
| TLD Nameserver | ❌ | ❌ | NS for domains only |
| Authoritative Nameserver | ❌ | ❌ | ✅ (zone file) |

---

## 2. Decision Logic

### What type of DNS record do I need?

```
What am I mapping?
├── Domain/subdomain → IPv4 address          → A record
├── Domain/subdomain → IPv6 address          → AAAA record
├── Domain → another domain (alias)          → CNAME record
│     ⚠️ Cannot use CNAME on apex domain    → Use ALIAS/ANAME (provider-specific)
├── Domain → mail server                     → MX record (must point to a name, not IP)
├── Zone delegation to sub-team/provider     → NS record
└── Zone metadata (primary NS, serial, etc.) → SOA record (exactly one per zone)
```

### Where is authority delegated?

```
Root Zone (IANA/ICANN)
    │── knows: NS for every TLD (.com, .io, .org...)
    │
    ▼
TLD Zone (Registry: Verisign, NIC.IO, PIR...)
    │── knows: NS for every registered 2nd-level domain
    │
    ▼
Domain Zone (Authoritative NS: Route53, DigitalOcean, GoDaddy...)
    │── knows: actual A, CNAME, MX, AAAA records
    │── can delegate subdomains to another zone via NS
    │
    ▼
Subdomain Zone (optional, e.g. maps.google.com → separate NS)
```

### Registrar vs DNS host — who does what?

| Responsibility | Registrar (e.g. GoDaddy) | DNS Host (e.g. Route 53) |
|---|---|---|
| Domain purchase & renewal | ✅ | ❌ |
| WHOIS / ownership records | ✅ | ❌ |
| Pushes NS records to TLD registry | ✅ | ❌ |
| Hosts zone file (A, MX, CNAME...) | ❌ (unless combined) | ✅ |
| Answers DNS queries | ❌ | ✅ |
| Edit DNS records / TTLs | ❌ | ✅ |

> You can register on GoDaddy and resolve on Route 53. GoDaddy exits the query path the moment NS delegation is done.

### When to lower TTL before a migration

```
Planning an IP change?
├── Lower TTL to 60–120 seconds several hours BEFORE the change
│     Why: resolvers cache the OLD TTL. TTL changes are not retroactive.
├── Make the IP change
├── Wait for old TTL to expire across the internet
└── Restore TTL to normal (3600+) after propagation confirms
```

---

## 3. Internal Working

### Full DNS resolution — step by step

**Scenario: Shwetangi types `cwvj.io` in her browser.**

```
Step 1:  Browser checks its own DNS cache
              → Hit? → Done (ms)
              → Miss? → Continue

Step 2:  OS checks /etc/hosts and local DNS cache
              → Hit? → Done
              → Miss? → Continue

Step 3:  OS sends query to configured DNS resolver
         (ISP resolver, or 8.8.8.8, or 1.1.1.1)

Step 4:  Resolver checks its cache for cwvj.io
              → Hit? → Returns cached A record immediately
              → Miss? → Begin recursive resolution

Step 5:  Resolver → Root Server (A–M.root-servers.net)
         Query:  "Who handles .io?"
         Answer: "Try a0.nic.io or b0.nic.io" (NS referral)

Step 6:  Resolver → .io TLD Nameserver (a0.nic.io)
         Query:  "Who handles cwvj.io?"
         Answer: "Try ns1.godaddy.com, ns2.godaddy.com" (NS referral)

Step 7:  Resolver → Authoritative NS (ns1.godaddy.com)
         Query:  "What is the A record for cwvj.io?"
         Answer: "15.202.3.55" (TTL: 3600)

Step 8:  Resolver caches result for TTL duration (3600s)

Step 9:  Resolver returns 15.202.3.55 to Shwetangi's OS

Step 10: OS passes IP to the browser

Step 11: Browser opens TCP connection → TLS handshake → HTTP(S) request → 🎉
```

**Total: ~12 steps on a cold cache. Typically sub-100ms due to resolver proximity.**

### What's inside a DNS zone file

```dns
$TTL 3600
@ IN SOA ns1.digitalocean.com. admin.kubernetes.io. (
          2025072401  ; Serial  — version number; increment on every change
          3600        ; Refresh — how often secondary NS checks for updates
          1800        ; Retry   — retry interval if refresh fails
          604800      ; Expire  — secondary NS stops serving after this (7 days)
          86400 )     ; Minimum TTL — used for negative (NXDOMAIN) responses

; Nameservers (must match what TLD has delegated)
     IN NS ns1.digitalocean.com.
     IN NS ns2.digitalocean.com.

; Apex A record
@    IN A  3.33.186.135

; Subdomains
www      IN CNAME kubernetes.io.
docs     IN A     52.219.165.51
blog     IN A     99.84.214.102
discuss  IN A     13.226.217.45

; Mail
         IN MX 10 mail.kubernetes.io.
mail     IN A     198.51.100.25
```

### How domain registration sets up the chain

```
1. You buy cwvj.io on GoDaddy
         │
2. GoDaddy assigns ns1.godaddy.com, ns2.godaddy.com
         │
3. GoDaddy sends NS record to NIC.IO (the .io registry):
         cwvj.io. NS ns1.godaddy.com.
         cwvj.io. NS ns2.godaddy.com.
         │
4. NIC.IO updates the .io TLD zone file
         │
5. Root servers already know .io → a0.nic.io (pre-existing, untouched)
         │
6. Chain is complete:
   Root → .io TLD → GoDaddy NS → your zone file → IP
```

### Anycast — why 13 root servers can handle the entire internet

```
"13 root servers" is a naming convention, not a machine count.

Each of the 13 labels (A.root-servers.net through M.root-servers.net)
is served by HUNDREDS of physical servers distributed globally.

Anycast routing: multiple servers share the same IP.
Your query automatically routes to the geographically nearest instance.

Result: root servers handle billions of queries/day with near-zero latency.
```

---

## 4. Hands-On

### Check your DNS resolver

```bash
# macOS
scutil --dns | grep nameserver

# Linux
cat /etc/resolv.conf
# nameserver 8.8.8.8

# Windows
ipconfig /all | findstr "DNS Servers"
```

### Manual DNS resolution with dig

```bash
# Full recursive lookup from root (bypasses resolver cache)
dig +trace docs.kubernetes.io

# Simple A record lookup
dig docs.kubernetes.io A

# With specific resolver
dig @8.8.8.8 docs.kubernetes.io

# Short output (just the IP)
dig +short docs.kubernetes.io

# Check TTL remaining
dig docs.kubernetes.io | grep -E "ANSWER|ttl"

# Query TLD nameservers for a domain's NS delegation
dig @a0.nic.io kubernetes.io NS

# Reverse DNS lookup (IP → hostname)
dig -x 3.33.186.135

# Check SOA record (zone metadata + serial number)
dig kubernetes.io SOA

# Check MX records
dig kubernetes.io MX

# Check AAAA (IPv6) records
dig kubernetes.io AAAA
```

### Inspect DNS resolution path

```bash
# dig +trace walks the full hierarchy and shows every delegation step
dig +trace cwvj.io

# Output shows:
# . 518399 IN NS a.root-servers.net.   ← root tells you .io nameservers
# io. 172800 IN NS a0.nic.io.          ← .io TLD tells you cwvj.io NS
# cwvj.io. 3600 IN NS ns1.godaddy.com. ← godaddy is authoritative
# cwvj.io. 3600 IN A 15.202.3.55       ← final answer
```

### Check existing priority classes (root nameservers)

```bash
# List the 13 root nameserver hostnames
dig . NS +short

# Verify anycast — query the same label from different locations
dig @a.root-servers.net . NS +short
```

### Validate a zone file locally

```bash
# Install bind-utils (Linux)
sudo apt install bind9utils -y

# Check zone file syntax
named-checkzone kubernetes.io /etc/bind/zones/kubernetes.io.zone

# Check reverse zone
named-checkzone 186.33.3.in-addr.arpa /etc/bind/zones/reverse.zone
```

### Sample DNS zone file (production-ready)

```dns
; kubernetes.io zone file
; Last updated: 2025-07-24 | Serial: 2025072401

$TTL 3600

; === SOA (required — exactly one per zone) ===
@ IN SOA ns1.digitalocean.com. admin.kubernetes.io. (
    2025072401  ; Serial
    3600        ; Refresh
    1800        ; Retry
    604800      ; Expire
    300 )       ; Minimum TTL (negative caching)

; === Nameservers (required — at least one NS) ===
@ IN NS ns1.digitalocean.com.
@ IN NS ns2.digitalocean.com.
@ IN NS ns3.digitalocean.com.   ; Third for redundancy

; === Apex domain ===
@ IN A 3.33.186.135
@ IN AAAA 2001:db8::1           ; IPv6

; === Subdomains ===
www     IN CNAME kubernetes.io.
docs    IN A     52.219.165.51
blog    IN A     99.84.214.102
discuss IN A     13.226.217.45
api     IN A     203.0.113.10

; === Mail ===
@ IN MX 10 mail.kubernetes.io.  ; Primary MX (lower = higher priority)
@ IN MX 20 mail2.kubernetes.io. ; Backup MX
mail    IN A 198.51.100.25
mail2   IN A 198.51.100.26

; === Subdomain zone delegation ===
; maps.kubernetes.io handled by a separate team/zone
maps    IN NS ns1.maps-dns.kubernetes.io.
maps    IN NS ns2.maps-dns.kubernetes.io.
```

---

## 5. Production Flow

### Full DNS architecture — domain lifecycle

```
┌──────────────────────────────────────────────────────────────────┐
│                        Internet                                  │
│                                                                  │
│   Root Zone (IANA/ICANN + Verisign)                             │
│   A–M.root-servers.net (13 names, ~1500 physical servers)       │
│   Stores: NS records for all TLDs                               │
│                    │                                             │
│                    │ delegates .io to                           │
│                    ▼                                             │
│   .io TLD Zone (NIC.IO)                                         │
│   a0.nic.io, b0.nic.io, c0.nic.io                              │
│   Stores: NS records for kubernetes.io, cwvj.io, etc.           │
│                    │                                             │
│                    │ delegates kubernetes.io to                 │
│                    ▼                                             │
│   Authoritative Zone (DigitalOcean)                             │
│   ns1.digitalocean.com, ns2.digitalocean.com                    │
│   Stores: A, AAAA, CNAME, MX for kubernetes.io                  │
└──────────────────────────────────────────────────────────────────┘

Query path for a cache miss:
User → Resolver → Root → .io TLD → DigitalOcean NS → Answer

GoDaddy (registrar) is NOT in the query path once NS is delegated.
```

### Multi-zone delegation pattern (Google-scale)

```dns
; google.com zone file — delegates subdomains to independent zones
maps    IN NS ns1.maps-dns.google.com.
mail    IN NS ns1.mail-dns.google.com.
cloud   IN NS ns1.cloud-dns.google.com.

; Each team owns their subdomain's zone independently
; google.com authoritative NS delegates responsibility, not queries
```

### Registrar + separate DNS host (common production setup)

```
[GoDaddy — Registrar]          [AWS Route 53 — DNS Host]
  - Bought myapp.com              - Hosts zone file
  - Updated NS to Route 53        - Serves A, CNAME, MX records
  - Handles renewals              - Answers all DNS queries
  - WHOIS info

Resolution path: Root → .com TLD → Route 53 → IP
GoDaddy: ← not in this path →
```

### TTL management for zero-downtime IP migration

```
T-24h:  Lower TTL from 3600 → 60 seconds
T-0:    Update A record to new IP
T+1m:   Max wait for all resolvers to refresh (60s TTL)
T+1h:   Verify traffic fully shifted to new IP
T+2h:   Restore TTL to 3600 seconds
```

---

## 6. Mistakes

### Mistake 1: Changing IP without lowering TTL first

**Symptom:** After an IP change, some users get the old IP for hours  
**Root cause:** Resolvers cached the old record with a 3600s TTL. TTL changes only apply to future caches — not retroactive.  
**Fix:** Lower TTL to 60–120s *several hours before* any planned IP change. Restore after migration.

---

### Mistake 2: Using CNAME on the apex domain

**Symptom:** Zone file validation fails. DNS provider throws error.  
**Root cause:** RFC prohibits CNAME on apex (`@`) because apex must have SOA and NS records, and CNAME cannot coexist with other record types.  
**Fix:**
```dns
; WRONG
@ IN CNAME otherdomain.com.   ; ❌ Invalid

; CORRECT — use A record for apex
@ IN A 203.0.113.10            ; ✅

; Or use ALIAS/ANAME (provider-specific feature that resolves the CNAME at the server)
; Supported by: Route 53 (ALIAS), Cloudflare (CNAME flattening), NS1
```

---

### Mistake 3: NS records in zone file don't match TLD delegation

**Symptom:** Domain resolves correctly via your registrar but fails for external users  
**Root cause:** The NS records inside your zone file differ from what the TLD registry has. Clients follow the TLD's NS records, not yours.  
**Fix:** Always ensure the NS records in your zone file match exactly what your registrar reported to the TLD. Check:
```bash
# What does .io TLD say?
dig @a0.nic.io kubernetes.io NS +short

# What does your zone file say?
dig @ns1.digitalocean.com kubernetes.io NS +short
# They must match
```

---

### Mistake 4: Forgetting to increment the SOA serial number

**Symptom:** Secondary nameservers don't pick up zone changes  
**Root cause:** Secondary NS uses the SOA serial to detect changes. If serial doesn't increase, it assumes nothing changed.  
**Fix:** Use `YYYYMMDDNN` format (e.g., `2025072401`). Increment the `NN` suffix for each change on the same day.

---

### Mistake 5: MX record points to an IP address

**Symptom:** Mail delivery fails; MX record ignored  
**Root cause:** RFC requires MX records to point to a hostname, not an IP. Mail servers won't follow a raw IP in MX.  
**Fix:**
```dns
; WRONG
@ IN MX 10 198.51.100.25     ; ❌ MX cannot point to IP

; CORRECT
@ IN MX 10 mail.example.com. ; ✅ Points to hostname
mail IN A 198.51.100.25      ; ✅ A record resolves the hostname
```

---

### Mistake 6: Assuming DNS "propagates" globally on its own timeline

**Symptom:** "DNS propagation is taking forever" — sometimes up to 48 hours  
**Root cause:** There's no central signal. Each resolver independently expires cached records per TTL. The 48-hour myth comes from old high-TTL defaults.  
**Fix:** Lower TTL before changes. After the change, use `dig +short` from multiple vantage points (or [dnschecker.org](https://dnschecker.org)) to verify propagation. Time = max(existing cached TTL).

---

## 7. Interview Answers

**Q: What is DNS and why does it exist?**

> "DNS stands for Domain Name System. It exists because humans remember names, not IP addresses, but computers need IP addresses to route traffic. DNS bridges this gap by acting as the internet's distributed phone book — when you type `kubernetes.io`, DNS resolves that name to an IP address like `3.33.186.135` so your browser knows which server to connect to. It's hierarchical and globally distributed, which means no single server holds all the answers — responsibility is delegated from root servers to TLD servers to authoritative nameservers, each owning only their slice of the namespace."

---

**Q: Walk me through what happens when a user types `docs.kubernetes.io` in their browser.**

> "First, the browser and operating system check their local DNS caches. If there's a valid cached entry, they use it and skip the rest. If not, the OS sends a query to its configured recursive resolver — typically your ISP's or a public one like 8.8.8.8. The resolver also checks its own cache. On a miss, it starts a recursive walk: it queries a root nameserver, which returns a referral to the `.io` TLD nameservers. The resolver then asks the `.io` nameservers for `kubernetes.io`, which returns a referral to DigitalOcean's nameservers. Finally, it queries DigitalOcean's nameservers directly, which return the A record for `docs.kubernetes.io` — the actual IP address. The resolver caches this for the TTL duration, returns the IP to the browser, and the browser initiates a TCP connection to load the page."

---

**Q: What's the difference between a domain registrar and a DNS hosting provider?**

> "These are two separate roles that are often confused. A registrar is where you buy and renew your domain — companies like GoDaddy, Namecheap, or AWS Route 53 when used as a registrar. The registrar is responsible for recording your ownership in the TLD registry and telling that registry which nameservers are authoritative for your domain. A DNS hosting provider is the one that actually runs those nameservers and stores your zone file — the A records, CNAME records, MX records, and so on. When someone queries your domain, the request goes to the DNS host, not the registrar. You can mix and match: register on GoDaddy, delegate to Route 53 for DNS hosting. Once delegation is done, GoDaddy is completely out of the resolution path."

---

**Q: What is the difference between an authoritative nameserver and a recursive resolver?**

> "An authoritative nameserver is the source of truth for a specific domain. It holds the actual zone file and returns definitive answers for records like A, CNAME, and MX. It does not cache or recurse — it just serves what's in its zone file. A recursive resolver, on the other hand, is a client-facing server that does the legwork of walking the DNS hierarchy on behalf of the user. It's the one that queries root servers, TLD servers, and authoritative servers in sequence. It caches results based on TTL to avoid repeating the work. Think of the authoritative server as the library, and the recursive resolver as the librarian who knows how to navigate it and remembers commonly requested books."

---

**Q: What is a DNS zone and what must every zone file contain?**

> "A DNS zone is a distinct administrative boundary within the DNS namespace — it's the portion of the hierarchy that a specific organization or provider is responsible for managing. For example, the `kubernetes.io` zone is managed by whoever runs DigitalOcean's nameservers on their behalf. Every valid zone file must contain exactly two things: a Start of Authority record, or SOA, which defines the primary nameserver for the zone, contact information, and timing metadata like serial numbers and refresh intervals; and at least one NS record, which specifies which nameservers are authoritative for this zone. Without these two, the zone is incomplete and cannot function correctly."

---

**Q: What is TTL and what happens if you change a DNS record without managing TTL first?**

> "TTL, or Time to Live, is the duration in seconds that a DNS resolver is allowed to cache a record before it must re-query the authoritative nameserver. If you change an A record today but the TTL was set to 86400 seconds — 24 hours — any resolver that cached it recently won't re-query for up to 24 hours. They'll keep sending traffic to the old IP. TTL changes are also not retroactive: if a resolver cached your record with a 24-hour TTL an hour ago, lowering the TTL now won't affect that resolver for another 23 hours. The best practice before any IP migration is to lower the TTL to 60 or 120 seconds several hours in advance, wait for all resolvers to pick up the short TTL, then make the change, then restore the TTL afterward."

---

**Q: Why can't you use a CNAME record on the apex domain?**

> "The DNS specification prohibits CNAME records from coexisting with any other record type on the same name. The apex domain — the bare domain like `example.com` — always requires SOA and NS records, which means it already has other record types. Since CNAME must be exclusive, putting a CNAME on the apex is technically invalid and will be rejected by compliant DNS servers. The workaround is to use an A record directly on the apex, or to use vendor-specific constructs like Route 53's ALIAS record or Cloudflare's CNAME flattening, which resolve the target and return a flat IP, effectively behaving like a CNAME without violating the RFC."

---

## 8. Debugging

### Fast diagnosis path for DNS issues

```
Can't reach a website by name?
         │
ping <domain>       →   "Name or service not known" / timeout?
         │
         ├── Name resolution failing → DNS issue
         │         │
         │         dig +short <domain>
         │         │
         │         ├── Returns IP → DNS works. Problem is network/server.
         │         │
         │         ├── Returns nothing → dig @8.8.8.8 <domain>
         │         │         │
         │         │         ├── 8.8.8.8 returns IP → Your resolver is broken
         │         │         │         Fix: change /etc/resolv.conf or router DNS
         │         │         │
         │         │         └── 8.8.8.8 also fails → authoritative NS issue
         │         │                   │
         │         │                   dig +trace <domain>
         │         │                   └── See where the chain breaks
         │         │
         │         └── Returns NXDOMAIN → domain doesn't exist at resolver
         │                   │
         │                   dig @<authoritative-ns> <domain>
         │                   ├── Returns IP → resolver has stale/missing cache
         │                   │    Fix: wait for TTL, or flush: sudo systemd-resolve --flush-caches
         │                   └── Also NXDOMAIN → record missing from zone file
         │
         └── Timeout → Network or firewall blocking UDP/TCP port 53
```

### Command toolkit

```bash
# 1. Basic resolution — does this domain have an A record?
dig +short docs.kubernetes.io

# 2. Full trace — walk the entire DNS hierarchy step by step
dig +trace docs.kubernetes.io

# 3. Force query through a specific resolver (bypass local cache)
dig @1.1.1.1 docs.kubernetes.io

# 4. Query directly against the authoritative NS
dig @ns1.digitalocean.com kubernetes.io A

# 5. Check TTL on a record
dig docs.kubernetes.io | awk '/ANSWER/,/AUTHORITY/' | grep -v "^;"

# 6. Check SOA serial (verify zone updates propagated)
dig kubernetes.io SOA +short
# Compare against secondary NS:
dig @ns2.digitalocean.com kubernetes.io SOA +short

# 7. Reverse lookup (IP → hostname)
dig -x 3.33.186.135 +short

# 8. Check NS delegation (what does the TLD say vs your zone file?)
dig @a0.nic.io kubernetes.io NS +short   # TLD's answer
dig @ns1.digitalocean.com kubernetes.io NS +short  # Your NS's answer
# They must match

# 9. Flush local DNS cache
# macOS:
sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder
# Linux (systemd-resolved):
sudo systemd-resolve --flush-caches
# Linux (nscd):
sudo service nscd restart

# 10. Validate zone file (requires bind9utils)
named-checkzone kubernetes.io /path/to/zone/file

# 11. Check propagation across global resolvers
# Use: https://dnschecker.org or:
for ns in 8.8.8.8 1.1.1.1 9.9.9.9 208.67.222.222; do
  echo -n "$ns: "
  dig @$ns docs.kubernetes.io +short
done
```

---

## 9. Kill Switch

**10-second recall — absolute minimum:**

```
DNS = name → IP translator, hierarchical, distributed, cached

Resolution path (cold cache):
Local → Resolver → Root → TLD → Authoritative NS → IP

Root:           stores NS for TLDs only
TLD:            stores NS for 2nd-level domains only
Authoritative:  stores actual records (A, CNAME, MX...)
Resolver:       caches + recurses on your behalf

Key record types:
  A      → IPv4 address
  AAAA   → IPv6 address
  CNAME  → alias (not on apex!)
  NS     → delegation pointer
  MX     → mail server (hostname, not IP)
  SOA    → zone metadata (exactly one per zone)

TTL = cache lifetime. Lower it before IP changes. Not retroactive.

Registrar  = where you BUY domain, pushes NS to TLD registry
DNS Host   = where you HOST zone file, answers queries
```

---

## 10. Appendix

### DNS record types cheatsheet

| Record | Purpose | Points To | Notes |
|---|---|---|---|
| `A` | IPv4 address | IP (e.g. `93.184.216.34`) | Most common |
| `AAAA` | IPv6 address | IPv6 (e.g. `2001:db8::1`) | — |
| `CNAME` | Alias | Another hostname | Cannot be on apex; no other records allowed on same name |
| `NS` | Nameserver delegation | Nameserver hostname | Used by root, TLD, and subdomain delegation |
| `MX` | Mail exchange | Hostname (not IP!) | Lower priority value = higher preference |
| `SOA` | Zone metadata | Inline fields | Exactly one per zone; required |
| `TXT` | Arbitrary text | String | SPF, DKIM, domain verification |
| `PTR` | Reverse DNS | Hostname | Used in `in-addr.arpa` zones |
| `SRV` | Service location | Host + port | Used by k8s headless services |

### Zone file field reference

```dns
$TTL <seconds>     ; Default TTL for all records in this zone

@ IN SOA <primary-ns>. <admin-email>. (
    YYYYMMDDNN   ; Serial: increment every change
    3600         ; Refresh: how often secondary checks primary
    1800         ; Retry: retry interval after failed refresh
    604800       ; Expire: secondary stops answering after this (7 days)
    300 )        ; Minimum TTL: negative cache TTL

; @ = apex (the zone itself, e.g. kubernetes.io)
; . after hostname = FQDN (no trailing dot = relative to zone)
```

### Public DNS resolvers reference

| Provider | IPv4 | IPv6 | Notes |
|---|---|---|---|
| Google | `8.8.8.8`, `8.8.4.4` | `2001:4860:4860::8888` | Global, fast |
| Cloudflare | `1.1.1.1`, `1.0.0.1` | `2606:4700:4700::1111` | Privacy-focused |
| Quad9 | `9.9.9.9` | `2620:fe::fe` | Security filtering |
| OpenDNS | `208.67.222.222` | `2620:119:35::35` | Configurable filtering |

### Root server operators

| Label | Operator | Anycast Instances (approx) |
|---|---|---|
| A | Verisign | 150+ |
| B | USC-ISI | 6 |
| C | Cogent | 10+ |
| D | UMD | 170+ |
| E | NASA | 25 |
| F | ISC | 300+ |
| G | DoD NIC | 6 |
| H | US Army | 8 |
| I | Netnod | 70+ |
| J | Verisign | 200+ |
| K | RIPE NCC | 80+ |
| L | ICANN | 200+ |
| M | WIDE Project | 20+ |

### Domain hierarchy vocabulary

```
www.kubernetes.io.
│   │           │ └─ Root (.) — implicit
│   │           └─── TLD — managed by Registry (NIC.IO)
│   └─────────────── 2nd-level domain — registered with Registrar
└─────────────────── Subdomain — defined in zone file by domain owner

FQDN = Fully Qualified Domain Name = name ending with trailing dot
```

### dig output anatomy

```
;; QUESTION SECTION:
;docs.kubernetes.io.       IN A       ← what we asked

;; ANSWER SECTION:
docs.kubernetes.io. 3600   IN A 52.219.165.51  ← the answer
    │               │         │  └── value
    │               │         └── record type
    │               └── TTL remaining (seconds)
    └── name queried

;; SERVER: 8.8.8.8#53    ← which resolver answered
;; Query time: 12 msec   ← latency
```

---
