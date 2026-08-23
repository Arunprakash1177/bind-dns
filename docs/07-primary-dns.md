# MODULE 07 — Primary Authoritative DNS

> Course: BIND DNS from Zero | Distro: AlmaLinux / Rocky / RHEL
> Builds on: Module 06 (working `named.conf`, ACL, listener configured)
> Builds: the `internal.example.com` zone, authoritative on **DNS01** — `192.168.10.10`

Everything so far has been preparation and generic infrastructure. This
is the module where DNS01 becomes authoritative for something that is
genuinely **yours** — a real zone, with real records, that you write by
hand from a blank file. This is also where Module 03's "domain vs zone"
and "authoritative vs recursive" distinctions stop being theory and
become lines in a config file you personally typed.

---

## 1. What this module covers

- Declaring a zone in `named.conf` — `type master` (Primary)
- Writing a zone file from a completely blank file
- Every field of the SOA record, explained individually
- NS, A, AAAA, CNAME, MX, TXT records — enough to build a working zone
  (full reference detail for every type is Module 10)
- TTL and serial numbers — what they are and why they matter
- Building the complete lab forward zone: DNS01, WEB01, DB01, GIT01
- Validating with `named-checkconf` and `named-checkzone`
- Querying your own new zone with `dig`

## 2. The lab domain and architecture for this module

```text
Zone: internal.example.com
Primary (this module): dns01.internal.example.com   192.168.10.10

Records we will create:
dns01   192.168.10.10   (this server itself)
web01   192.168.10.20
db01    192.168.10.30
git01   192.168.10.40
```

This exact host list is used consistently through the rest of the
course — Module 09 builds the reverse zone for these same IPs, Module
14 replicates this zone to DNS02, and Module 29's final lab extends it
further.

---

## 3. Declaring the zone in `named.conf`

Open `/etc/named.conf` (the file you built in Module 06) and add a new
`zone {}` block, after the existing `zone "." IN { ... };` block:

```text
zone "internal.example.com" IN {
    type master;
    file "internal.example.com.zone";
    allow-transfer { none; };
};
```

**Every directive explained:**

- `zone "internal.example.com" IN` — declares a zone for this exact
  name, in the `IN` (Internet) class (essentially always `IN` in
  modern practice, as noted in Module 04).
- `type master;` — this is the **Primary** server for this zone (BIND's
  configuration syntax still uses the historical term `master`, even
  though the surrounding documentation and community increasingly say
  "primary" — you will see both terms used interchangeably in real
  configs and official docs, and should recognize them as synonyms).
  This means: the zone data is edited directly here, on this server,
  and this server is the origin of truth for it.
- `file "internal.example.com.zone";` — the zone file's name, resolved
  relative to the `directory "/var/named";` setting from Module 06's
  `options {}` block — so the real path is
  `/var/named/internal.example.com.zone`.
- `allow-transfer { none; };` — explicitly denies zone transfers (AXFR,
  Module 15) for now. We deliberately set this restrictively now and
  will *explicitly* open it only to DNS02's specific IP in Module 14 —
  never leave zone transfer open to everyone, a security point revisited
  in depth in Module 15/17.

---

## 4. Writing the zone file — from a blank file

Create the file:

```bash
sudo vi /var/named/internal.example.com.zone
```

We'll build this up piece by piece, explaining every line, then show
the complete file at the end of this section.

### The `$TTL` directive

```text
$TTL 86400
```

**What it does:** sets the **default** TTL (Module 03) applied to any
record in this file that doesn't specify its own explicit TTL. `86400`
seconds = 24 hours — a reasonable default for a stable internal lab
zone. (Individual records can still override this with their own
explicit TTL value if needed — not done in this module, but the syntax
exists and is used occasionally in real production zones for records
that need to change faster or slower than the zone default.)

### The SOA record — field by field

```text
@       IN      SOA     dns01.internal.example.com. admin.internal.example.com. (
                            2026082201  ; serial
                            3600        ; refresh
                            900         ; retry
                            604800      ; expire
                            86400 )     ; minimum
```

This is the single most information-dense line in any zone file. Every
field, explained:

- **`@`** — a special shorthand meaning "the zone's own origin name" —
  here, that resolves to `internal.example.com.` itself, taken from the
  zone's declaration in `named.conf`. Using `@` instead of spelling out
  the full name is standard practice and avoids repeating the zone name
  throughout the file.
- **`IN`** — class, Internet, as before.
- **`SOA`** — record type: Start Of Authority. Every zone file must
  have exactly one SOA record, and it must be the **first** record in
  the file (after `$TTL`).
- **`dns01.internal.example.com.`** — the **primary name server** for
  this zone (called the "MNAME" field internally). Note the **trailing
  dot** — this makes it a fully qualified name (Module 03's FQDN
  concept made syntactically literal). **Forgetting this trailing dot
  is one of the single most common zone-file mistakes** — without it,
  BIND would interpret this as a *relative* name and silently append
  the zone's own origin to it again, producing something like
  `dns01.internal.example.com.internal.example.com.` — wrong, and not
  always obviously wrong at a glance. Always double-check trailing dots
  on every fully-qualified name in a zone file.
- **`admin.internal.example.com.`** — the zone administrator's contact
  ("RNAME" field), written in a slightly unusual format: it represents
  an email address, but with the `@` replaced by a `.` — so this
  represents `admin@internal.example.com`, not a hostname called
  `admin`. (If the real email's local part itself contains a literal
  dot, like `first.last@example.com`, that dot must be escaped as
  `\.` — a detail worth knowing exists even though our simple `admin@`
  address doesn't need it.)
- **`serial`** (`2026082201`) — a number that must **increase** every
  time you change the zone file. This is how secondary servers (Module
  14) know new data is available, and how caching resolvers understand
  data may have changed. **Convention used in this course:**
  `YYYYMMDDNN` — today's date plus a two-digit revision counter for
  that day (`01` for the first change today, `02` for a second change
  the same day, etc.). This is one of several common conventions
  (others simply increment a plain integer by 1 each time) — the only
  hard *requirement* BIND enforces is that each new value must be
  numerically larger than the previous one; the `YYYYMMDDNN` convention
  is popular specifically because it's human-readable and
  self-documenting about *when* a change was made.
- **`refresh`** (`3600` = 1 hour) — how often a **secondary** server
  should check the primary's SOA serial to see if a zone transfer is
  needed. Fully relevant starting Module 14.
- **`retry`** (`900` = 15 minutes) — if a secondary's refresh check
  *fails* (primary unreachable), how long to wait before retrying.
- **`expire`** (`604800` = 7 days) — if a secondary **cannot reach the
  primary at all** for this entire duration, it should stop treating
  its (now-stale) copy of the zone as authoritative and stop answering
  for it — a safety mechanism against serving indefinitely stale data.
- **`minimum`** (`86400` = 24 hours) — historically the default TTL;
  in modern BIND/DNS usage this field's meaning was redefined (via RFC
  2308) to specifically control the TTL applied to **negative caching**
  — how long resolvers may cache an `NXDOMAIN` ("this name doesn't
  exist") response for names in this zone. Covered again in Module 13.

**The parentheses `( ... )` spanning multiple lines** are purely a
readability convenience — BIND's zone-file parser allows a record to
span multiple lines when wrapped in parentheses, which is standard
practice for SOA specifically given how many fields it has. The
semicolons after each value here are **comment markers** (zone-file
comments, not the statement-terminator semicolons from `named.conf`'s
different syntax — don't confuse the two languages), documenting what
each bare number means for the next human who reads this file — which
is essential, because the raw numbers alone are not self-explanatory.

### NS records

```text
        IN      NS      dns01.internal.example.com.
```

**What it does:** declares which server(s) are authoritative for this
zone. Note the blank space where the name would go — when a record's
name field is left blank, BIND implicitly reuses the name from the
**previous** record (here, that's `@`, from the SOA record above) —
this is a common zone-file shorthand you'll see throughout real-world
files, not a mistake. We'll add a second NS line for DNS02 in Module
14, once it exists.

### A records — the actual hosts

```text
dns01   IN      A       192.168.10.10
web01   IN      A       192.168.10.20
db01    IN      A       192.168.10.30
git01   IN      A       192.168.10.40
```

Straightforward: name (relative to the zone origin — `dns01` here means
`dns01.internal.example.com.`, since there's no trailing dot and it's
not `@`), class, type, and the IPv4 address. **Note the deliberate
contrast with the SOA/NS lines above:** these names have **no** trailing
dot, because they're *meant* to be relative — BIND appends the zone
origin (`internal.example.com.`) automatically. This relative-vs-fully-
qualified distinction, and exactly when each is correct, is precisely
what the earlier SOA warning about the trailing dot was preparing you
for — get comfortable telling these two cases apart on sight.

### A CNAME record — an alias

```text
www     IN      CNAME   web01.internal.example.com.
```

**What it does:** `www.internal.example.com` is not a separate host —
it's an **alias** pointing at `web01`. A query for `www`'s A record
gets resolved by BIND first finding this CNAME, then following it to
`web01`'s actual A record, and typically returning both pieces of
information in the answer (visible directly in `dig` output — you'll
see this in the verification section below). Full CNAME depth,
including important restrictions on where CNAMEs can and can't be used,
is Module 10.

### An MX record — mail routing

```text
        IN      MX      10 dns01.internal.example.com.
```

**What it does:** declares that mail for `internal.example.com` should
be delivered to `dns01` (a simplification for lab purposes — a real
production zone would typically point MX at a dedicated mail host, not
the DNS server itself). The `10` is a **priority** value — lower
numbers are preferred; if multiple MX records exist, mail servers try
the lowest-priority one first, falling back to higher-priority values
only if delivery fails. Full detail in Module 10.

### A TXT record

```text
        IN      TXT     "internal.example.com lab zone - BIND DNS course"
```

**What it does:** a free-form text record — here just a descriptive
label for the zone, but in real production use, TXT records commonly
carry domain-ownership verification tokens or email-authentication
data (SPF/DKIM), previewed in Module 04 and detailed in Module 10.

---

## 5. The complete zone file

The full file — also saved in this repository at
[`configs/forward-zone/internal.example.com.zone`](../configs/forward-zone/internal.example.com.zone)
for reference:

```text
$TTL 86400
@       IN      SOA     dns01.internal.example.com. admin.internal.example.com. (
                            2026082201  ; serial
                            3600        ; refresh
                            900         ; retry
                            604800      ; expire
                            86400 )     ; minimum

        IN      NS      dns01.internal.example.com.
        IN      MX      10 dns01.internal.example.com.
        IN      TXT     "internal.example.com lab zone - BIND DNS course"

dns01   IN      A       192.168.10.10
web01   IN      A       192.168.10.20
db01    IN      A       192.168.10.30
git01   IN      A       192.168.10.40

www     IN      CNAME   web01.internal.example.com.
```

**Fix ownership and permissions** — directly applying Module 05's
lesson, before this file will even load:

```bash
sudo chown root:named /var/named/internal.example.com.zone
sudo chmod 640 /var/named/internal.example.com.zone
```

---

## 6. Validation — `named-checkconf` and `named-checkzone`

Never skip this. Two separate tools, two separate jobs:

```bash
sudo named-checkconf
```
Validates `named.conf` syntax — confirms your new `zone {}` block is
syntactically well-formed. Silent output = success (Module 06).

```bash
sudo named-checkzone internal.example.com /var/named/internal.example.com.zone
```

**What it does:** parses the **zone file itself** — a completely
different, more detailed check than `named-checkconf -z` performs,
specifically validating record syntax, SOA correctness, and zone-file
structure.

**Expected output on success:**
```
zone internal.example.com/IN: loaded serial 2026082201
OK
```
Note it echoes back the **serial number it read** — a useful, quick way
to confirm you're looking at the version of the file you think you are,
especially valuable later once Module 14's secondary is comparing
serials across two servers.

**Deliberately break something to see a real failure** — remove the
trailing dot from the SOA's MNAME field and re-run:
```
dns01.internal.example.com.internal.example.com is not qualified and
zone internal.example.com/IN: has 0 SOA records...
```
(the exact wording varies by BIND version, but the message will point
you at the malformed name) — fix it back and confirm clean output
again before proceeding.

---

## 7. Apply and verify

```bash
sudo systemctl reload named
```
**Note this is `reload`, not `restart`** — directly applying Module
05's lesson: you added zone *content*, not a core `options {}` setting,
so `reload` is correct and sufficient here, and it's what you'll use
for essentially every future zone-file change in this course.

**Confirm the zone loaded, via logs:**
```bash
journalctl -u named -n 20 --no-pager
```
Look for a line confirming the zone loaded successfully, e.g.
`zone internal.example.com/IN: loaded serial 2026082201`.

### Query it with `dig`

```bash
dig @192.168.10.10 dns01.internal.example.com
dig @192.168.10.10 web01.internal.example.com
dig @192.168.10.10 www.internal.example.com
dig @192.168.10.10 internal.example.com MX
dig @192.168.10.10 internal.example.com NS
dig @192.168.10.10 internal.example.com SOA
```

**Expected ANSWER SECTION for `web01`:**
```
;; ANSWER SECTION:
web01.internal.example.com. 86400 IN   A       192.168.10.20
```

**Expected ANSWER SECTION for `www` — notice the CNAME chain, exactly
as predicted in section 4 above:**
```
;; ANSWER SECTION:
www.internal.example.com. 86400 IN     CNAME   web01.internal.example.com.
web01.internal.example.com. 86400 IN   A       192.168.10.20
```

**Confirm authoritative status specifically** — check the `flags:` line
in the header:
```
;; flags: qr aa rd ra;
```
The **`aa`** flag (Authoritative Answer) appearing here is the concrete,
literal proof of Module 03's abstract "authoritative" concept — this
response isn't cached or relayed from somewhere else, DNS01 is the
actual source of truth for this data, and BIND is telling you so
explicitly in the response flags.

---

## What I Learned

- How to declare a `type master` (Primary) zone in `named.conf`,
  including why `allow-transfer { none; };` is the correct, deliberate
  starting posture
- How to write a zone file from a blank file: `$TTL`, the full SOA
  record field by field (MNAME, RNAME, serial, refresh, retry, expire,
  minimum), NS, A, CNAME, MX, and TXT records
- The critical importance of trailing dots on fully-qualified names
  versus relative names, and how to tell which is correct in each
  position
- Why serial numbers must always increase, and the `YYYYMMDDNN`
  convention
- The distinct jobs of `named-checkconf` (config syntax) versus
  `named-checkzone` (zone file content/structure)
- How to read the `aa` (Authoritative Answer) flag in `dig` output as
  literal, direct confirmation of authoritative status

## Commands to Remember

```bash
sudo vi /etc/named.conf                      # add zone {} block
sudo vi /var/named/internal.example.com.zone  # write the zone file
sudo chown root:named /var/named/internal.example.com.zone
sudo chmod 640 /var/named/internal.example.com.zone
sudo named-checkconf
sudo named-checkzone internal.example.com /var/named/internal.example.com.zone
sudo systemctl reload named
journalctl -u named -n 20 --no-pager
dig @192.168.10.10 web01.internal.example.com
dig @192.168.10.10 internal.example.com SOA
```

## Practical Lab

1. Add the `zone "internal.example.com" IN { type master; ... };` block
   to `named.conf` and validate with `named-checkconf`.
2. Write the complete zone file from section 5, exactly as shown, then
   fix ownership/permissions.
3. Validate with `named-checkzone internal.example.com
   /var/named/internal.example.com.zone` and confirm `OK`.
4. `reload` named and confirm the zone loaded via `journalctl`.
5. Query every record you created (`dns01`, `web01`, `db01`, `git01`,
   `www`, the zone's `NS`, `MX`, `SOA`, and `TXT` records) with `dig
   @192.168.10.10` and confirm each returns the expected data.
6. Confirm the `aa` flag is present in each authoritative response.
7. Increment the serial number, change `web01`'s IP to `192.168.10.21`
   as a drill, reload, and confirm `dig` reflects the new value — then
   change it back to `.20` (increment the serial again) to match the
   rest of this course.

## Troubleshooting Exercise

**Scenario:** after writing the zone file and reloading, `dig
@192.168.10.10 web01.internal.example.com` returns `SERVFAIL` (recall
Module 04's precise distinction: this means an error, not "doesn't
exist").

1. **Symptom:** `SERVFAIL` on every query for this zone.
2. **Diagnosis:**
   ```bash
   sudo named-checkzone internal.example.com /var/named/internal.example.com.zone
   journalctl -u named -n 30 --no-pager
   ```
3. **Commands:** `named-checkzone`, `journalctl -u named`, `ls -l
   /var/named/internal.example.com.zone`
4. **Root cause (most common, in order of likelihood):**
   (a) a missing trailing dot in the SOA MNAME/RNAME, producing a
   malformed name and zero valid SOA records; (b) the zone file has
   wrong ownership/permissions (Module 05's exact lesson) so `named`
   (running as the unprivileged `named` user) can't read it at all; (c)
   a missing semicolon or unbalanced parenthesis in the SOA block.
5. **Fix:** `named-checkzone`'s output will point at the specific line
   — fix the identified issue (correct the trailing dot, run `chown
   root:named` + `chmod 640`, or fix the syntax), then re-validate.
6. **Verification:** `named-checkzone` reports `OK` with the correct
   serial; `sudo systemctl reload named`; `journalctl -u named` shows
   the zone loaded; `dig` returns `NOERROR` with the `aa` flag present.

## Interview Questions

- **Walk through every field of an SOA record and what it controls.**
  Short answer: primary NS name, admin contact, serial (change
  counter), refresh (how often secondaries check for updates), retry
  (how soon to retry after a failed check), expire (how long a
  secondary may serve stale data before giving up), and minimum
  (negative-caching TTL). Detailed: together these fields entirely
  govern how zone data propagates from a primary to its secondaries and
  how long various failure/staleness conditions are tolerated before
  the system takes protective action. Real-world example: a team about
  to perform planned zone-data maintenance might temporarily lower
  `refresh`/`retry` beforehand so secondaries pick up the eventual
  change faster, then restore the original values afterward — a real,
  if relatively advanced, operational technique directly built on
  understanding these fields individually.

- **Why must the serial number always increase, and what convention did
  you use?**
  Short answer: secondaries and caching resolvers use the serial purely
  as a "is there something newer" comparison — it must strictly
  increase for that comparison to mean anything. Detailed: this course
  uses `YYYYMMDDNN` for human readability, but the only hard rule BIND
  enforces is numeric increase from the previous value; accidentally
  *decreasing* it (e.g. after restoring an old backup zone file without
  bumping the serial past what secondaries already have) causes
  secondaries to believe their existing copy is already current and
  skip the transfer entirely. Real-world example: this exact scenario —
  restoring from an old backup — is one of Module 27's explicit backup/
  restore considerations, precisely because of this serial-number
  trap.

- **What does the `aa` flag in a `dig` response actually prove, and why
  does it matter?**
  Short answer: it proves the responding server holds the zone's actual
  source-of-truth data and isn't relaying a cached or forwarded answer.
  Detailed: a caching resolver or a server merely forwarding your query
  elsewhere will return an answer without `aa` set, even if the data
  itself is correct — the flag specifically distinguishes "this exact
  server is authoritative for this exact zone" from "this server
  happens to know the answer." Real-world example: when troubleshooting
  a discrepancy between what two different DNS servers return for the
  same name, checking whether `aa` is set on each response tells you
  immediately which one (if either) you should actually trust as
  ground truth, versus which might just be serving a stale cached
  value.

## Expected Result

DNS01 is now genuinely, verifiably **authoritative** for
`internal.example.com` — a real zone you wrote by hand, correctly
declared, validated, loaded, and confirmed with the `aa` flag in live
`dig` output. `dns01`, `web01`, `db01`, and `git01` all resolve
correctly, `www` correctly aliases to `web01` via CNAME, and the zone's
NS/MX/SOA/TXT records are all in place.

This zone file's structure — SOA, NS, A, CNAME, MX, TXT — is exactly
what **MODULE 08 — DNS Zone Files Deep Dive** now unpacks line by line
in even greater depth, focusing specifically on `$TTL`, `$ORIGIN`, and
every serial-number convention in use in real production environments.

---

Say **NEXT** to continue to Module 08 — DNS Zone Files Deep Dive.
