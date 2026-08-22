# MODULE 03 — DNS Fundamentals

> Course: BIND DNS from Zero | Distro: AlmaLinux / Rocky / RHEL
> Builds on: Modules 01–02 (DNS01 prepared, networking fluency established)

No BIND installation yet — this module is entirely conceptual, and
deliberately so. Every command you'll run starting Module 04 only makes
sense once you can explain, from memory, what a resolver, a zone, and
an authoritative server actually are, and how a query travels from a
client's laptop to an answer. Skipping this module is the single most
common reason people can *operate* BIND by copy-pasting configs without
being able to *explain* or *troubleshoot* it.

---

## 1. What DNS is

DNS (Domain Name System) is a distributed, hierarchical naming system
that translates human-friendly names (`www.example.com`) into the
identifiers machines actually use to communicate (IP addresses like
`93.184.216.34`, or other record data like mail server names).

It is **distributed**: no single server holds all DNS data for the
entire internet. It is **hierarchical**: authority for different parts
of the namespace is delegated downward, from root, to top-level domain,
to the organization that owns a specific domain.

## 2. Why DNS exists

Computers route traffic using numeric addresses (IP addresses). Humans
don't memorize numbers well, and — critically — the IP address behind a
name can **change** (server migrations, load balancer changes, cloud
provider changes) without the name changing. DNS provides:

1. **Human-usable naming** instead of raw IP addresses
2. **Indirection** — the ability to change what's behind a name without
   every client needing to know
3. **Load distribution and geographic routing** — a single name can
   resolve to different answers depending on the resolver, enabling
   CDNs and load balancing
4. **Service discovery patterns** — MX records for "where does mail for
   this domain go," SRV records for "where does this specific service
   live" (Module 10)

**Production framing:** in a DevOps context, DNS is the layer that lets
you point `web01.internal.example.com` at a new IP after a server
rebuild without touching a single application config file, load
balancer rule, or hardcoded connection string elsewhere in your
infrastructure — that's the entire practical value proposition.

---

## 3. Core vocabulary

Work through these definitions carefully — they get reused, precisely,
in every subsequent module.

### Domain name
A name in the DNS hierarchy, e.g. `example.com`, `internal.example.com`.

### FQDN (Fully Qualified Domain Name)
A domain name that specifies its exact position in the DNS hierarchy,
unambiguous with no relative interpretation needed — e.g.
`dns01.internal.example.com.` (note: technically, a fully qualified
name ends in a trailing dot representing the DNS root, though it's
usually omitted in casual use and in most tool output).

### Hostname
The label identifying a specific machine, usually the leftmost part of
an FQDN — `dns01` is the hostname portion of `dns01.internal.example.com`.

### Domain vs Zone — the distinction that trips up almost everyone

This is the single most important distinction in this module.

- A **domain** is a name and everything logically underneath it in the
  naming hierarchy — a namespace concept.
- A **zone** is an administrative unit: a specific, distinct portion of
  the DNS namespace that one particular set of authoritative servers is
  responsible for, and that is defined by one particular zone file.

**Why they're not the same thing:** a single domain can be split across
**multiple zones** through delegation. `example.com` might be one zone,
while `internal.example.com` — even though it's "under" `example.com`
in the namespace — can be **delegated** to a completely different set
of authoritative servers and defined by its own, separate zone file.
That's exactly our lab setup: `internal.example.com` is its own zone,
served by DNS01/DNS02, entirely independent of whoever might run the
parent `example.com` zone (which, in a real production setup, could be
a completely different team or even a different organization).

**Interview framing to remember:** *"A domain is a name; a zone is who's
responsible for answering for a portion of that name."*

### Resolver
Software that performs DNS lookups on behalf of a client — either a
**stub resolver** (the lightweight OS-level component, e.g. `glibc`'s
resolver reading `/etc/resolv.conf`, that just forwards the question to
a configured DNS server) or a **recursive resolver** (a full DNS server
that does the actual multi-step work of walking the hierarchy to find
an answer — Module 11 covers this in depth).

### Authoritative DNS server
A server that holds the actual, original zone data for a zone and can
answer queries about it with a definitive, authoritative answer — as
opposed to a resolver that's just caching or relaying someone else's
answer.

### Root servers
The 13 logical root server clusters (`a.root-servers.net` through
`m.root-servers.net`, each actually backed by many physical/anycast
servers worldwide) that are authoritative for the DNS root zone
(`.`) — they don't know the IP for `example.com` directly, but they
know which servers are authoritative for the `.com` TLD.

### TLD servers
Servers authoritative for a Top-Level Domain (`.com`, `.org`, `.net`,
country codes like `.in`, `.uk`). They don't know `example.com`'s
records directly, but they know which servers are authoritative for
`example.com` specifically (this is a **delegation**, mechanically
implemented via NS records — Module 10).

### DNS query / DNS response
A query is a request for specific record data about a specific name
(e.g. "what's the A record for `web01.internal.example.com`?"). A
response is the server's answer, which may be a successful answer, a
referral to another server, or an error (like NXDOMAIN — "this name
doesn't exist").

### Recursive query vs iterative query
- A **recursive query** says, in effect: "give me the final answer, do
  whatever work is necessary, I don't want to be handed intermediate
  referrals." Clients (stub resolvers) always send recursive queries to
  their configured DNS server.
- An **iterative query** says: "give me the best answer you currently
  have, even if it's just a referral to someone else who might know
  more." This is what a recursive resolver sends to root/TLD/
  authoritative servers on the client's behalf — those servers don't do
  recursive work themselves, they just refer the resolver onward.

**This is the core mechanical distinction of Module 11** — internalize
it now: *clients ask recursively; recursive resolvers ask everyone else
iteratively.*

### Caching
Once a recursive resolver gets an answer, it stores it temporarily
(governed by the record's TTL) so that repeated queries for the same
name don't require re-walking the entire hierarchy. Covered in depth in
Module 13.

### TTL (Time To Live)
A value, in seconds, attached to every DNS record, specifying how long
a resolver is allowed to cache that record before it must be
considered stale and re-fetched from the authoritative server. Short
TTLs mean faster propagation of changes but more query load on
authoritative servers; long TTLs mean less load but slower propagation
if you need to change something quickly (e.g. during an incident).

---

## 4. The complete resolution process, end to end

Walk through what actually happens when a client looks up
`web01.internal.example.com`, assuming **nothing** is cached anywhere
yet (a "cold" lookup — the worst case, and the case that teaches you
the most):

```
Client (stub resolver)
 ↓  "I want the A record for web01.internal.example.com — recursive query"
Recursive Resolver  (e.g. your ISP's resolver, or your own BIND server in recursive mode)
 ↓  "Who's authoritative for the root?" — resolver already knows this, it's hardcoded (root hints)
Root Server
 ↓  "I don't know web01... but here's who's authoritative for .com" (referral, iterative)
TLD Server (.com)
 ↓  "I don't know web01... but here's who's authoritative for example.com" (referral, iterative)
   [in a real chain, there could be another referral step here for
    internal.example.com if it's delegated as a separate zone —
    exactly our lab setup]
Authoritative DNS Server (for internal.example.com — this is DNS01!)
 ↓  "web01.internal.example.com is 192.168.10.20" (authoritative answer)
Recursive Resolver
 ↓  caches the answer for the record's TTL, then relays it back
Client
 ↓  now has 192.168.10.20, and can make its actual connection
```

**Key things to notice in this diagram:**

1. The client only ever talks to **one** server (its configured
   recursive resolver) — it does none of the walking itself. That's the
   entire point of a recursive resolver existing.
2. The recursive resolver does **all** the iterative work, potentially
   several round trips, entirely invisible to the client.
3. Only the **final** server in the chain (the authoritative server for
   the specific zone containing the answer) gives a definitive,
   authoritative response. Every server before that gives a referral.
4. On a **warm** lookup (something already cached, within its TTL), the
   recursive resolver skips straight from "client asks" to "resolver
   answers from cache" — no walk at all. This is why caching (Module
   13) matters so much for real-world DNS performance.

### Where our lab fits into this diagram

In our lab, DNS01 will eventually play **two different roles**
simultaneously (a very common, if not universally recommended,
real-world pattern, discussed further in Module 11):
- **Authoritative** for the `internal.example.com` zone — it's the
  final, definitive answer source for names in that zone.
- **Recursive resolver + forwarder** for CLIENT01 and other internal
  hosts — it does the "walk the hierarchy" work (or forwards that work
  to `8.8.8.8`/`1.1.1.1`, Module 12) for anything *outside* its own
  authoritative zone, like public internet domains.

Keeping straight *which role BIND is playing for a given query* is
essential — Module 06 onward, the `recursion` setting and zone
declarations in `named.conf` are literally how you tell BIND which role
to play for which names.

---

## 5. Primary vs Secondary vs Authoritative vs Recursive vs Caching vs Forwarding — the full picture

These terms get used loosely in casual conversation, but they answer
**different questions** and are not mutually exclusive — a single BIND
server is often several of these at once, for different zones or
purposes.

| Term | Question it answers | Notes |
|---|---|---|
| **Authoritative** | "Do you hold the actual source-of-truth data for this zone?" | Opposite of "just caching/relaying someone else's answer" |
| **Primary** (formerly "master") | "Is this the server where zone data is directly edited?" | The zone file lives here; changes originate here |
| **Secondary** (formerly "slave") | "Did this server get its zone data via transfer from a primary?" | Read-only copy, kept in sync via AXFR/IXFR (Module 14–15) |
| **Recursive** | "Will this server do the full walk-the-hierarchy work on a client's behalf?" | Governed by `recursion yes/no;` in `named.conf` |
| **Caching** | "Does this server remember previous answers to avoid repeating work?" | Nearly every recursive resolver is also a caching resolver — they go together in practice |
| **Forwarding** | "Does this server hand off certain queries to another specific DNS server instead of doing its own recursive walk?" | An alternative to full recursion for external/unknown names (Module 12) |

**Worked example using our actual lab:** DNS01 will be:
- **Authoritative** (and specifically **Primary**) for
  `internal.example.com`
- **Recursive** and **Caching** for anything CLIENT01 asks that isn't
  in that zone
- Possibly configured to **forward** those non-authoritative queries to
  `8.8.8.8`/`1.1.1.1` rather than doing full root-to-TLD recursion
  itself (a very common real-world choice, covered in Module 12)

DNS02 will be:
- **Authoritative** and specifically **Secondary** for
  `internal.example.com` (gets its zone data from DNS01 via zone
  transfer, Module 14)

**Interview framing:** *"Primary/secondary describes where zone data
originates. Authoritative/recursive describes what kind of answer a
server is allowed to give. Forwarding/caching describes how a
non-authoritative query gets resolved. A real server's role is usually
a specific combination of all three axes, not a single label."*

---

## What I Learned

- The precise definitions of domain, zone, FQDN, resolver, authoritative
  server, root/TLD server — vocabulary reused in every later module
- Why "domain" and "zone" are **not** synonyms, and how delegation
  splits a namespace across multiple zones/authorities
- The complete cold-lookup resolution path from client to authoritative
  answer, and where caching short-circuits it
- The difference between a recursive query (client → resolver) and an
  iterative query (resolver → everyone else)
- That Primary/Secondary, Authoritative/Recursive, and
  Caching/Forwarding are three **independent axes**, not synonyms —
  and how DNS01 and DNS02 will each occupy a specific combination of
  them in our lab

## Commands to Remember

No new commands this module — this is a concept-only module. Use
Module 04's `dig` tooling next to observe every concept covered here
directly against real, live DNS infrastructure.

## Practical Lab

No hands-on lab yet — instead, complete this written exercise before
moving on (you'll validate your answers empirically in Module 04):

1. Draw the resolution diagram from section 4 from memory, without
   looking back at this document.
2. For our lab, write out explicitly which role(s) — Authoritative,
   Primary, Secondary, Recursive, Caching, Forwarding — apply to DNS01
   and which apply to DNS02.
3. Explain, in your own words, why `internal.example.com` can be its
   own zone even though it's "underneath" `example.com` in the
   namespace.

## Troubleshooting Exercise

This module has no hands-on troubleshooting (no software installed
yet) — but here's a conceptual diagnostic scenario to reason through,
which foreshadows real Module 21 content:

**Scenario:** A colleague says "DNS is broken, `internal.example.com`
names aren't resolving for anyone." You ask: "is this affecting
internet domains too, or just our internal zone?" They say "just our
internal zone, google.com works fine."

1. **What does this immediately tell you, conceptually, before running
   any commands?** The recursive/forwarding path to the outside world
   is intact (proven by `google.com` working) — so the problem is
   isolated to the **authoritative** side: either DNS01's zone data for
   `internal.example.com`, or something specific to how that zone is
   being served/delegated. It is very unlikely to be a general
   network, firewall, or "DNS server is completely down" issue, because
   *some* DNS resolution is clearly working.
2. **What would you check first, and why?** Whether `named` is
   authoritative and actually loaded the `internal.example.com` zone
   correctly — you're narrowing from "all of DNS" to "one zone" before
   touching a single config file, purely from the symptom description.

This kind of reasoning — narrowing scope from symptoms *before*
touching commands — is exactly the skill Module 21 formalizes into a
full methodology.

## Interview Questions

- **What's the difference between a domain and a zone?**
  Short answer: a domain is a name and its namespace; a zone is the
  administrative unit of who's authoritative for (a portion of) that
  namespace. Detailed: one domain can be split into multiple zones
  through delegation — the parent zone holds an NS record pointing at
  the child zone's authoritative servers, and from that point on, the
  child zone is served entirely independently. Real-world example:
  `example.com` and `internal.example.com` can be run by entirely
  different teams, on entirely different DNS software, because
  they're separate zones despite `internal.example.com` being "part
  of" the `example.com` domain namespace.

- **Explain the difference between a recursive query and an iterative
  query.**
  Short answer: a recursive query demands a final answer; an iterative
  query accepts a referral to another server. Detailed: clients (stub
  resolvers) only ever issue recursive queries to their configured DNS
  server; that server then does the actual multi-hop work by issuing
  iterative queries to root, TLD, and authoritative servers on the
  client's behalf, each of which is free to reply with just a referral
  rather than a final answer. Real-world example: this split is exactly
  why running a public, fully open recursive resolver is dangerous
  (Module 11/17) — you're offering to do potentially expensive
  multi-hop work on behalf of anyone who asks, which is precisely the
  mechanism abused in DNS amplification attacks.

- **What does TTL control, and what's the trade-off in setting it low
  vs high?**
  Short answer: TTL controls how long a resolver may cache a record
  before re-querying the authoritative server. Detailed: a low TTL
  means changes propagate faster but authoritative servers see more
  query volume; a high TTL reduces load and speeds up cached lookups
  but delays how quickly clients see an update. Real-world example:
  teams commonly lower a record's TTL in advance of a planned
  migration (e.g. a data center cutover) specifically so that, when the
  actual change happens, clients pick up the new value quickly instead
  of continuing to hit the old IP for hours.

## Expected Result

You can now explain, from memory and without notes, what happens
between a client typing a hostname and getting back an IP address —
including every hop in the hierarchy, the recursive/iterative
distinction, and how caching and TTL affect that process. You can also
correctly place DNS01 and DNS02's future roles onto the
Primary/Secondary, Authoritative/Recursive, Caching/Forwarding axes
before a single line of BIND configuration exists.

This conceptual foundation is what makes Module 04's `dig` output, and
every `named.conf` directive from Module 06 onward, make immediate
sense rather than feeling like magic syntax to memorize.

---

Say **NEXT** to continue to Module 04 — DNS Tools.
