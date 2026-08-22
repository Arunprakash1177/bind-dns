# MODULE 04 — DNS Tools

> Course: BIND DNS from Zero | Distro: AlmaLinux / Rocky / RHEL
> Builds on: Modules 01–03 (server ready, networking fluency, DNS concepts)

Still no BIND installation — and that's deliberate. You are about to
spend the rest of this course reading `dig` output every single time
you change a config file. If you can't read that output fluently
*before* you have your own server to test against, you'll spend Module
07 onward debugging your own confusion about `dig`, layered on top of
debugging your actual BIND config. So we build tool fluency now, against
**public, already-working DNS infrastructure** — no room for "is my
server broken or am I misreading the tool" ambiguity.

---

## 1. What this module covers

- `dig` — the primary tool for the rest of this course, in depth
- `host` and `nslookup` — lighter alternatives, and why `dig` is
  generally preferred professionally
- `getent hosts` — the NSS-aware lookup you already used in Module 01
- `resolvectl` — where relevant on systemd-resolved systems
- Reverse lookups (`dig -x`)
- Querying a specific server directly, bypassing your configured
  resolver
- `dig +trace` — watching the full recursive walk from Module 03 happen
  live, in real output

## 2. Installing the tools

On AlmaLinux/Rocky/RHEL, `dig`, `host`, and `nslookup` all ship in the
**`bind-utils`** package (note: this is a *client-tools* package,
completely separate from the `bind` package that provides the actual
`named` server daemon — installing `bind-utils` now does not install a
DNS server, and Module 05 will install `bind` separately).

```bash
sudo dnf install -y bind-utils
```

**Verify:**
```bash
dig -v
```
**Expected output (version string, exact numbers will vary):**
```
DiG 9.16.23-RH
```

**Ubuntu/Debian note:** the equivalent package is `dnsutils`
(`sudo apt install -y dnsutils`), and confusingly, the actual DNS
server package is named `bind9` rather than `bind` — a naming
difference worth remembering before Module 05.

---

## 3. `dig` — the core tool, in depth

`dig` ("Domain Information Groper") is the standard, professional-grade
DNS query tool. Unlike `nslookup` (interactive-oriented, somewhat
inconsistent output) or `host` (terse, good for scripting but limited
detail), `dig` shows you the **entire** DNS message — every section,
every flag — which is exactly what you need for real troubleshooting.

### The most basic query

```bash
dig google.com
```

**Full expected output, annotated:**

```
; <<>> DiG 9.16.23-RH <<>> google.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 47213
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
;; QUESTION SECTION:
;google.com.                   IN      A

;; ANSWER SECTION:
google.com.             168     IN      A       142.250.195.174

;; Query time: 14 msec
;; SERVER: 192.168.10.10#53(192.168.10.10)
;; WHEN: Sat Aug 22 15:04:11 IST 2026
;; MSG SIZE  rcvd: 55
```

### Every section, explained

**HEADER line** — `opcode: QUERY, status: NOERROR, id: 47213`
- `opcode: QUERY` — a standard query (other opcodes exist, like
  `UPDATE` for Dynamic DNS, rarely seen in normal troubleshooting)
- `status: NOERROR` — the query succeeded. Other common statuses:
  `NXDOMAIN` (name doesn't exist), `SERVFAIL` (server-side failure,
  often a misconfiguration), `REFUSED` (server declined to answer —
  Module 17's ACLs directly control this)
- `id: 47213` — a random query ID, used to match responses to requests
  and as a (weak) defense against off-path response spoofing

**flags** — `qr rd ra`
- `qr` — Query Response (this message is a response, not a query)
- `rd` — Recursion Desired (the client asked for a recursive answer)
- `ra` — Recursion Available (the server is willing to do recursive
  work) — **if `ra` is missing from a response, the server you queried
  is not offering recursion to you**, which is either intentional
  (Module 11's security posture) or a misconfiguration, depending on
  context
- `QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1` — a count of how
  many records appear in each section below

**OPT PSEUDOSECTION** — EDNS0 information (Module 02's mention of
extended UDP size limits) — `udp: 1232` here means this resolver
advertises it can handle UDP responses up to 1232 bytes before
requiring TCP fallback.

**QUESTION SECTION** — restates exactly what was asked: name, class
(`IN` = Internet — essentially always `IN` in modern practice; other
classes are historical/unused), and type (`A`).

**ANSWER SECTION** — the actual answer(s):
```
google.com.             168     IN      A       142.250.195.174
```
Read left to right: name, **TTL (168 seconds remaining)**, class, type,
data. This TTL is a live, decrementing value — if you re-run the exact
same query a few seconds later against a resolver that's caching the
answer, you'll see the number drop; it directly demonstrates Module 03's
caching/TTL concept in real output.

**AUTHORITY SECTION** — (empty here) would list NS records for the
zone, typically shown in referral responses or NXDOMAIN responses
rather than successful answers.

**ADDITIONAL SECTION** — (the `1` counted, though not always printed as
visible text unless it's a real record — here it's the OPT
pseudo-record itself) supplementary records the server includes
proactively, e.g. the A record for an NS server named in the AUTHORITY
section, saving the client a follow-up query.

**Footer line:**
```
;; Query time: 14 msec
;; SERVER: 192.168.10.10#53(192.168.10.10)
;; WHEN: Sat Aug 22 15:04:11 IST 2026
;; MSG SIZE  rcvd: 55
```
- `Query time` — round-trip latency for this specific query
- `SERVER` — which resolver `dig` actually asked (your configured
  `/etc/resolv.conf` server, unless overridden with `@`, below)
- `WHEN` — local timestamp
- `MSG SIZE rcvd` — total response size in bytes — relevant to Module
  02's UDP-size/truncation discussion

---

## 4. Querying specific record types

```bash
dig A google.com
dig AAAA google.com
dig MX google.com
dig NS google.com
dig TXT google.com
dig SOA google.com
```

**What each type means (full detail in Module 10, brief preview here):**
- `A` — IPv4 address (the default type if you don't specify one)
- `AAAA` — IPv6 address
- `MX` — Mail eXchanger, where mail for this domain should be delivered
- `NS` — Name Server, which servers are authoritative for this zone
- `TXT` — free-form text, commonly used for domain verification (e.g.
  Google Workspace, SPF/DKIM email authentication records)
- `SOA` — Start Of Authority, metadata about the zone itself (Module 08
  covers every field in depth)

**Example: `dig SOA google.com`, ANSWER SECTION:**
```
google.com.             60      IN      SOA     ns1.google.com. dns-admin.google.com. 761234567 900 900 1800 60
```
This single line packs the zone's primary NS, admin contact (dots
instead of `@`, historically), serial number, and the refresh/retry/
expire/minimum timers — you'll build one of these yourself from scratch
in Module 07/08.

---

## 5. `+short` — the scriptable form

```bash
dig +short google.com
```
**Expected output:**
```
142.250.195.174
```
**What it does:** strips away every section except the raw answer data
— exactly what you want when piping `dig` output into a script, a
health-check (Module 25/26), or when you just want a fast answer
without reading a full message. You'll use `+short` constantly in
automation later in this course.

```bash
dig +short NS google.com
```
```
ns1.google.com.
ns2.google.com.
ns3.google.com.
ns4.google.com.
```

---

## 6. Querying a specific server directly — `@`

By default, `dig` asks whatever resolver is configured in
`/etc/resolv.conf`. Override this with `@<server>`:

```bash
dig @8.8.8.8 google.com
dig @1.1.1.1 google.com
```

**Why this matters enormously for the rest of this course:** once DNS01
is running BIND, you will constantly need to ask "is *this specific
server* answering correctly," independent of whatever your local
machine happens to have configured. This is how you test DNS01 directly
even before you've pointed any client at it:

```bash
dig @192.168.10.10 web01.internal.example.com
```

This single pattern — `dig @<target-server> <name>` — is the single
most-used command for the rest of this entire course, starting Module 07.

**Verify the SERVER line changes accordingly:**
```
;; SERVER: 8.8.8.8#53(8.8.8.8)
```
confirms you actually reached the server you targeted, not your default
resolver — always sanity-check this line when debugging.

---

## 7. `dig +trace` — watch the entire hierarchy walk, live

This command makes Module 03's resolution diagram real and visible.

```bash
dig +trace google.com
```

**What it does:** instead of asking your configured resolver to do
recursive work and just hand you the final answer, `dig +trace`
performs the **iterative walk itself** — starting from the root, and
printing every referral step along the way.

**Abbreviated expected output:**
```
; <<>> DiG 9.16.23-RH <<>> +trace google.com
;; global options: +cmd
.                       518400  IN      NS      a.root-servers.net.
.                       518400  IN      NS      b.root-servers.net.
[... more root servers ...]
;; Received 811 bytes from 192.168.10.10#53(192.168.10.10) in 12 ms

com.                    172800  IN      NS      a.gtld-servers.net.
com.                    172800  IN      NS      b.gtld-servers.net.
[... more .com TLD servers ...]
;; Received 1176 bytes from 198.41.0.4#53(a.root-servers.net) in 34 ms

google.com.             172800  IN      NS      ns1.google.com.
google.com.             172800  IN      NS      ns2.google.com.
[... more google.com NS records ...]
;; Received 168 bytes from 192.5.6.30#53(a.gtld-servers.net) in 41 ms

google.com.             300     IN      A       142.250.195.174
;; Received 59 bytes from 216.239.32.10#53(ns1.google.com) in 22 ms
```

**Read this against Module 03's diagram directly:** each `;; Received
... from ...` line is one hop — root, then `.com` TLD, then
`google.com`'s own authoritative servers, each one referring `dig`
onward until the final authoritative answer arrives from
`ns1.google.com` itself. This is the *exact* iterative process a real
recursive resolver performs on a client's behalf, just made visible.

**Use this constantly once your own zone exists** — `dig +trace
web01.internal.example.com` will show you whether delegation to your
own authoritative servers is even working correctly, independent of any
caching.

---

## 8. Reverse lookups — `dig -x`

```bash
dig -x 8.8.8.8
```

**What it does:** performs a **PTR** lookup — given an IP address, find
the associated hostname. Internally, `dig -x` is shorthand: it
automatically constructs the correct `in-addr.arpa` reverse-lookup name
(`8.8.8.8` becomes `8.8.8.8.in-addr.arpa`) and queries for a PTR
record — you'll build these reverse zones by hand in Module 09.

**Expected output, ANSWER SECTION:**
```
8.8.8.8.in-addr.arpa.  86400   IN      PTR     dns.google.
```

---

## 9. `host` and `nslookup` — lighter alternatives

### `host`

```bash
host google.com
```
**Expected output:**
```
google.com has address 142.250.195.174
google.com has IPv6 address 2404:6800:4009:825::200e
google.com mail is handled by 10 smtp.google.com.
```
Terser than `dig`, good for a fast human-readable check or simple
scripting, but shows less diagnostic detail (no flags, no TTL by
default, no clear section breakdown) — this is exactly why `dig` is
preferred for real troubleshooting, where those details matter.

### `nslookup`

```bash
nslookup google.com
```
**Expected output:**
```
Server:         192.168.10.10
Address:        192.168.10.10#53

Non-authoritative answer:
Name:   google.com
Address: 142.250.195.174
```
`nslookup` predates `dig`, has an interactive mode
(`nslookup` with no arguments drops you into a prompt), and is
available cross-platform (including Windows) — useful to know because
you'll sometimes be troubleshooting from a Windows client where `dig`
isn't available by default. **Professionally, `dig` is preferred on
Linux/Unix systems** for its completeness; `nslookup`'s output format
has also been explicitly noted in its own man page as "not guaranteed
stable," which is a real reason serious tooling avoids parsing it.

---

## 10. `getent hosts` — the NSS-aware lookup (revisited from Module 01)

```bash
getent hosts google.com
```
**What it does, and why it's different from `dig`:** `getent hosts`
doesn't talk to DNS directly — it goes through **NSS (Name Service
Switch)**, following whatever order `/etc/nsswitch.conf` specifies
(typically `files` first, i.e. `/etc/hosts`, then `dns`). This is the
**same resolution path real applications use** when they call standard
library functions like `getaddrinfo()`.

**Why this distinction matters for troubleshooting (a preview of Module
21):** if `dig` successfully returns an answer but an actual
application still fails to connect using that hostname, `getent hosts`
is often the missing diagnostic step — it can reveal that `/etc/hosts`
has a stale conflicting entry that's shadowing DNS entirely, something
`dig` alone would never show you, because `dig` bypasses NSS and
`/etc/hosts` completely, talking to a DNS server directly.

---

## 11. `resolvectl` (systemd-resolved systems)

On systems running `systemd-resolved` (more common by default on
Ubuntu Desktop and some Ubuntu Server configurations than on
RHEL-family minimal installs):

```bash
resolvectl status
resolvectl query google.com
```
`resolvectl status` shows per-interface DNS configuration and which
resolver is actually in effect; `resolvectl query` performs a lookup
through `systemd-resolved`'s own resolution path (which may itself be
caching, separately from BIND). If you're on a RHEL-family minimal
install as this course assumes, you likely won't have
`systemd-resolved` active at all — running `resolvectl status` in that
case will simply report the service isn't running, which is expected
and fine.

---

## 12. Practical DNS troubleshooting exercises

Run each of these against public infrastructure and interpret the
result before moving to Module 05:

```bash
# 1. Confirm NOERROR vs NXDOMAIN
dig this-domain-absolutely-does-not-exist-12345.com
```
**Expected `status:`** `NXDOMAIN` — confirm you understand this means
"the name genuinely doesn't exist," distinct from `SERVFAIL` (a server
problem) — a distinction that matters immensely in Module 21.

```bash
# 2. Confirm recursion is being offered
dig @1.1.1.1 google.com | grep flags
```
**Expected:** flags include `ra` — Cloudflare's `1.1.1.1` is a public
recursive resolver, so it should always offer recursion to anyone.

```bash
# 3. Compare TTL decrementing on a repeated cached query
dig +short google.com; sleep 5; dig google.com | grep -A1 "ANSWER SECTION"
```
Observe the TTL number in the second query is a few seconds lower than
it would have been on the very first cold query — direct, visible proof
of caching in action.

```bash
# 4. Trace a domain end-to-end
dig +trace github.com
```
Identify, in the output, exactly which line represents the root
referral, which represents the TLD referral, and which represents the
final authoritative answer.

```bash
# 5. Reverse-lookup a well-known IP
dig -x 1.1.1.1
```

---

## What I Learned

- How to read every section of `dig` output: HEADER, flags, QUESTION,
  ANSWER, AUTHORITY, ADDITIONAL, and the footer (`Query time`, `SERVER`,
  `WHEN`, `MSG SIZE`)
- The difference between `NOERROR`, `NXDOMAIN`, `SERVFAIL`, and
  `REFUSED` statuses
- How to query specific record types (`A`, `AAAA`, `MX`, `NS`, `TXT`,
  `SOA`) and a specific server directly with `@`
- How to watch the full recursive/iterative resolution walk from
  Module 03 happen live with `dig +trace`
- How to perform reverse lookups with `dig -x`
- Why `getent hosts` (NSS-aware) can disagree with `dig` (talks to DNS
  directly), and why that distinction matters for real application
  troubleshooting
- Why `dig` is professionally preferred over `nslookup`/`host` for
  serious troubleshooting

## Commands to Remember

```bash
dig <name>
dig <TYPE> <name>
dig +short <name>
dig @<server> <name>
dig +trace <name>
dig -x <ip>
host <name>
nslookup <name>
getent hosts <name>
resolvectl status
resolvectl query <name>
```

## Practical Lab

1. Run `dig google.com` and manually label every section of the output
   without referring back to this document.
2. Run `dig +short MX google.com` and explain what the leading number
   before each mail server name represents (priority — lower is
   preferred; full detail in Module 10).
3. Run `dig @8.8.8.8 cloudflare.com` and `dig @1.1.1.1 cloudflare.com`
   — confirm the `SERVER:` line in each output matches the server you
   targeted.
4. Run `dig +trace` against a domain of your choice and write down,
   hop by hop, which servers responded at each stage.
5. Run `dig -x` against your own home/office public IP (find it first
   with `dig +short myip.opendns.com @resolver1.opendns.com`) and see
   whether a PTR record exists for it — many residential IPs have none,
   which is itself a useful, common real-world result to recognize.

## Troubleshooting Exercise

**Scenario:** A teammate runs `dig internal-app.example.com` and gets
`status: SERVFAIL`. They ask if the domain "doesn't exist."

1. **Symptom:** `SERVFAIL` status.
2. **Diagnosis:** explain, before running anything, why this is
   categorically different from `NXDOMAIN` — `SERVFAIL` means the
   server *tried* to resolve the name and hit an error doing so (often:
   the authoritative server for that zone is unreachable, misconfigured,
   or the zone itself has a data error), not that the name is confirmed
   not to exist.
3. **Commands to actually run:**
   ```bash
   dig internal-app.example.com
   dig +trace internal-app.example.com
   dig NS example.com
   ```
4. **Root cause (illustrative, not this specific domain):** commonly,
   the zone's authoritative servers are unreachable, or (in a scenario
   we'll build ourselves starting Module 07) a zone file has a syntax
   error preventing `named` from loading it at all.
5. **Fix:** depends on root cause — this exercise is about correctly
   narrowing the diagnosis category before jumping to a fix, which is
   the actual skill being tested here.
6. **Verification:** re-run `dig` and confirm `status: NOERROR` with a
   valid ANSWER section once the underlying issue is resolved.

## Interview Questions

- **What's the difference between `NXDOMAIN` and `SERVFAIL`?**
  Short answer: `NXDOMAIN` means the name definitively doesn't exist;
  `SERVFAIL` means the server encountered an error trying to resolve it
  and couldn't give a definitive answer either way. Detailed:
  `NXDOMAIN` is itself a valid, authoritative, cacheable answer (this
  is "negative caching," covered in Module 13); `SERVFAIL` usually
  indicates something is broken — an unreachable authoritative server,
  a zone-loading error, or a DNSSEC validation failure — and is not
  something a resolver should cache the same way. Real-world example:
  after a typo'd zone file causes `named` to fail to load a zone
  (Module 21 covers this exact failure), every query for names in that
  zone returns `SERVFAIL`, not `NXDOMAIN` — recognizing this
  distinction immediately tells an engineer to check the DNS server's
  own health and logs rather than assume the record was simply never
  created.

- **Why does `dig +trace` sometimes give a different, "closer to
  ground truth" answer than a normal `dig` query?**
  Short answer: `+trace` bypasses your resolver's cache entirely and
  performs the full iterative walk itself, so it reflects the current
  live authoritative data, whereas a normal query might return a cached
  (potentially stale, within its TTL) answer. Detailed: this makes
  `+trace` extremely useful right after making a DNS change, when you
  need to confirm the authoritative servers are serving the *new* data,
  without waiting for caches elsewhere to expire. Real-world example:
  immediately after updating a record on your primary DNS server,
  `dig +trace` against your own authoritative servers confirms the
  change took effect at the source, even while other resolvers around
  the world are still serving the old cached value until their TTL
  expires.

- **Why is `dig` generally preferred over `nslookup` for professional
  troubleshooting?**
  Short answer: `dig` exposes the full DNS message (flags, all
  sections, TTLs, exact server queried) in a stable, scriptable format,
  while `nslookup`'s output is terser and explicitly not guaranteed
  stable across versions. Detailed: real troubleshooting frequently
  depends on details `nslookup` doesn't surface by default — whether
  recursion was available (`ra` flag), the exact TTL remaining, which
  section a record appeared in (answer vs authority vs additional).
  Real-world example: diagnosing a slave DNS server that isn't
  receiving zone updates (Module 14/21) requires comparing SOA serial
  numbers precisely via `dig SOA`, which is fast and exact — `nslookup`
  can technically do this too, but with meaningfully more friction and
  less scriptable output.

## Expected Result

You can now run `dig` confidently against any DNS server (yours or
someone else's), correctly interpret every section of its output,
distinguish `NOERROR`/`NXDOMAIN`/`SERVFAIL`/`REFUSED`, watch a full
recursive resolution walk with `+trace`, and know when `getent hosts`
might disagree with raw `dig` output and why.

You are now fully equipped to install and verify your own DNS server.
**MODULE 05 — Install BIND** installs the actual `named` daemon on
DNS01, and every verification step from here on will lean directly on
the tools built in this module.

---

Say **NEXT** to continue to Module 05 — Install BIND.
