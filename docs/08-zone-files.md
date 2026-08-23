# MODULE 08 — DNS Zone Files Deep Dive

> Course: BIND DNS from Zero | Distro: AlmaLinux / Rocky / RHEL
> Builds on: Module 07 (`internal.example.com` zone written and loaded)

Module 07 got a working zone file onto disk. This module goes back over
that same file — and the zone-file format in general — with a much
finer-toothed comb: every directive, every serial-number convention
used in real production environments, and the subtle mistakes that
`named-checkzone` catches versus the ones it silently lets through
because they're technically valid syntax but wrong intent.

---

## 1. What this module covers

- `$TTL` and `$ORIGIN` — the two directives that govern an entire file
- Every record type used so far (SOA, NS, A, AAAA, CNAME, MX, TXT, PTR)
  revisited specifically from a **zone-file syntax** angle, not a
  "what does this record mean" angle (that's Module 10)
- Serial number conventions used in real production teams, compared
- Why serial numbers matter beyond "just increment it"
- Absolute vs relative names inside a zone file, formalized

## 2. `$TTL` — revisited in full

You used this in Module 07:
```text
$TTL 86400
```

**What it precisely controls:** the *default* TTL applied to any
resource record in the file that does not specify its own. It must
appear **before** the first record in the file (by convention, and in
most BIND versions, as the very first line).

**You can override it per-record**, which real production zones do
constantly — most commonly to give one specific record a **much
shorter** TTL than the rest of the zone, usually in preparation for a
planned change:

```text
$TTL 86400
@       IN      SOA     ...

web01   300     IN      A       192.168.10.20
db01            IN      A       192.168.10.30
```

Here, `web01`'s record explicitly sets its own TTL to `300` (5 minutes)
— overriding the file's `$TTL 86400` default — while `db01` (with no
explicit TTL field) inherits the file-wide default of 86400 seconds.

**Real production pattern worth memorizing:** before a planned
migration of a specific host, an engineer will often lower just that
record's TTL days in advance (giving caches worldwide time to expire
their long-TTL copies and pick up the new short TTL), perform the
migration, then raise the TTL back to normal afterward once the new
value has propagated and stabilized. This is a genuinely common,
practical DevOps technique — not just textbook trivia.

## 3. `$ORIGIN` — controlling what "relative" means

Module 07 briefly touched on relative vs absolute names. `$ORIGIN` is
the directive that explicitly controls what "relative" resolves
against.

**Where it comes from implicitly (what you already relied on in Module
07 without seeing it written):** when a zone is loaded via `named.conf`
(`zone "internal.example.com" IN { ... };`), BIND **implicitly sets**
`$ORIGIN` to that zone's name (`internal.example.com.`) for the entire
file, from the start — which is exactly why writing `web01` (no
trailing dot) in Module 07's zone file correctly became
`web01.internal.example.com.` without you ever typing `$ORIGIN`
explicitly.

**You can also set or change it explicitly, mid-file:**
```text
$ORIGIN internal.example.com.
dns01   IN      A       192.168.10.10

$ORIGIN corp.internal.example.com.
laptop01   IN      A       192.168.10.90
```
After the second `$ORIGIN` line, any unqualified name (like `laptop01`
here) resolves against the *new* origin
(`laptop01.corp.internal.example.com.`), not the zone's original name.

**Why this matters, and when you'd actually use it:** this lets a
single zone file serve multiple logical sub-namespaces under one
zone without needing a fully separate delegated zone (Module 03's
domain-vs-zone distinction) for each one. This course's lab keeps
things simple with one `$ORIGIN` for the whole file (the implicit one
from `named.conf`), but recognizing this directive — and that it's the
*mechanism* controlling every relative-name interpretation in the
file — is essential for reading real-world zone files that use it more
elaborately.

**The `@` symbol, precisely defined now:** `@` is shorthand for
"whatever `$ORIGIN` currently is" at that point in the file — which is
why it meant `internal.example.com.` on the SOA line in Module 07 (the
implicit origin from `named.conf`, still in effect at the top of the
file), and would mean something different after an explicit mid-file
`$ORIGIN` change like the example above.

---

## 4. Every record type, revisited from the zone-file-syntax angle

This section is deliberately narrower than Module 10's full reference
— here we only care about **exactly how each type is written** in a
zone file, field by field, syntactically. What each type *means* and
when to use it is Module 10's job.

### `$TTL`
Directive, not a record — covered above.

### `$ORIGIN`
Directive, not a record — covered above.

### SOA
```text
@   IN  SOA   <MNAME> <RNAME> ( <serial> <refresh> <retry> <expire> <minimum> )
```
Exactly one per zone file, must be the first record. Full field-by-
field breakdown already given in Module 07 — not repeated here, but
worth re-reading now that `$ORIGIN`/`@` are precisely defined, since
both the MNAME and RNAME fields' correctness depend on understanding
exactly what "fully qualified" (trailing dot, absolute) versus
"relative to `$ORIGIN`" means.

### NS
```text
    IN  NS    <nameserver-fqdn>
```
One line per authoritative server for the zone. Name field
conventionally left blank (inherits `@` from context, per Module 07),
though writing `@   IN   NS   ...` explicitly is equally valid and
some teams prefer the explicit form for clarity.

### A
```text
<host>   IN  A     <ipv4-address>
```
One line per IPv4 address. A single hostname *can* have multiple A
records (multiple IPs) — BIND will return all of them, and which one a
client's stub resolver actually uses first is up to that client
(commonly round-robin or first-in-list, depending on the OS) — a simple
form of load distribution, distinct from more sophisticated DNS-based
load balancing setups.

### AAAA
```text
<host>   IN  AAAA  <ipv6-address>
```
Identical structure to A, for IPv6. Not used in this course's lab
(IPv4-only, per Module 02), but syntactically trivial to add if needed
— e.g. `dns01   IN   AAAA   2001:db8::10`.

### CNAME
```text
<alias>   IN  CNAME  <canonical-target-fqdn>
```
**Critical zone-file-syntax rule, not previously stated explicitly:** a
name that has a CNAME record **cannot have any other record type**
at the same name — no A, no MX, no TXT, nothing else, ever, for that
exact name. This is a hard DNS protocol rule, not a BIND-specific
restriction. `named-checkzone` will often (though not always, depending
on exact conditions) catch a clear violation of this, but it's better
to simply never attempt it — if you find yourself wanting to add a
second record type to a name that already has a CNAME, that's a signal
the design itself needs rethinking (usually: the *target* should have
the additional record, not the alias). Also note the target must be
written as a fully-qualified name with a trailing dot if it's meant
absolutely (as in `web01.internal.example.com.` in Module 07's file) —
the same trailing-dot rule as everywhere else.

### MX
```text
    IN  MX    <priority> <mailserver-fqdn>
```
Priority is a plain integer (lower preferred); the mail server name
should be an FQDN pointing at a name that itself has an A record — an
MX record must never point at a CNAME (another protocol-level rule,
not just a style preference).

### TXT
```text
    IN  TXT   "<any free-form text>"
```
Must be wrapped in double quotes. TXT records have a per-string length
limit (255 characters for a single quoted string, historically) — very
long TXT values (common with some DKIM keys) are written as multiple
adjacent quoted strings that BIND concatenates, e.g.
`"first part" "second part"` — worth knowing this pattern exists even
though this course's lab TXT record is short enough not to need it.

### PTR
```text
<reversed-last-octet>   IN  PTR   <fqdn>
```
Used exclusively in reverse zones — full treatment, including exactly
how the reversed-octet name is constructed, is Module 09, immediately
next. Mentioned here only for completeness of the record-type list.

---

## 5. Serial number conventions compared

Module 07 used `YYYYMMDDNN`. This is extremely common, but not
universal — know the alternatives, because you will encounter all of
them in real environments and need to recognize what you're looking
at.

| Convention | Example | Pros | Cons |
|---|---|---|---|
| `YYYYMMDDNN` | `2026082201` | Human-readable; self-documents *when* a change happened; up to 99 changes/day | Fails if you make a 100th change in one day (rare in practice); requires discipline to actually update the date portion correctly rather than copy-pasting a stale one |
| Plain incrementing integer | `47`, `48`, `49` ... | Simplest possible; never runs into a daily-change-count ceiling | Tells you nothing about *when* a change happened just by looking at it; easy to lose track of the "current" value across a team without checking `dig SOA` |
| Unix timestamp | `1755878400` | Always increasing by definition, extremely unlikely to ever collide | Not human-readable at a glance; a bit unconventional |

**The only rule BIND actually enforces, regardless of which convention
you pick:** the new value must be numerically greater than the previous
one (with one subtlety: DNS serial number comparison uses "serial
number arithmetic," RFC 1982, which handles wraparound near the 32-bit
integer boundary — an edge case essentially irrelevant to a lab or even
most real production zones, but worth knowing exists if you ever
encounter a zone with a genuinely enormous serial history).

**Team convention matters more than which specific scheme you pick** —
what actually causes real incidents is a team member manually editing a
zone file and **forgetting to bump the serial at all**. Since Module
07's zone file, and this course generally, edits files by hand, build
the habit *now*: every single zone-file edit gets a serial bump, no
exceptions, before you ever save-and-reload. (Module 25's automation
work later actually scripts this exact step to remove the human-error
risk entirely — worth remembering as motivation for why that
automation is worth building.)

---

## 6. Absolute vs relative names — the full, formal rule

Now that `$ORIGIN` is explicit, here's the precise, complete rule for
every name that appears in a zone file:

- A name ending in a **trailing dot** (`dns01.internal.example.com.`)
  is **absolute** — used exactly as written, nothing appended.
- A name **without** a trailing dot (`web01`, `dns01.internal`) is
  **relative** — BIND appends the current `$ORIGIN` to it.
- `@` alone means "the current `$ORIGIN`, exactly."
- A blank name field means "reuse the name from the previous record" —
  this isn't relative/absolute at all, it's a separate shorthand.

**Deliberately construct and observe a mistake, to make this concrete:**

```bash
sudo vi /var/named/internal.example.com.zone
# temporarily change the web01 line to:
#   web01   IN   A   192.168.10.20
#     -->  web01.internal.example.com   IN   A   192.168.10.20   (no trailing dot!)
```

```bash
sudo named-checkzone internal.example.com /var/named/internal.example.com.zone
```
**What happens:** this does **not** produce an error — it's
syntactically valid. But because there's no trailing dot, `$ORIGIN`
gets appended, silently creating a record for
`web01.internal.example.com.internal.example.com.` — an unintended,
"orphan" name that exists in the zone but is not what you meant to
create, and the *original* `web01` record you actually wanted no longer
exists.

```bash
dig @192.168.10.10 web01.internal.example.com
```
**Result:** `NXDOMAIN` — proof the intended record is now missing,
even though `named-checkzone` reported no error at all, because nothing
here violates zone-file *syntax* — only your *intent*. **This is
precisely why the trailing-dot habit from Module 07 must become
automatic** — no validation tool can catch this class of mistake for
you, because from BIND's perspective, you asked for exactly what you
got.

Revert the change back to the correct `web01   IN   A   192.168.10.20`
form, re-validate, reload, and confirm `dig` succeeds again before
moving on.

---

## What I Learned

- `$TTL` sets the file-wide default TTL, and can be overridden per
  record — a real production pattern before planned migrations
- `$ORIGIN` controls what relative names resolve against, is set
  implicitly by the zone's name in `named.conf`, and can be changed
  explicitly mid-file
- `@` means "the current `$ORIGIN`," precisely
- The zone-file syntax for every record type used so far, plus the
  hard protocol rule that a CNAME name cannot coexist with any other
  record type at that same name
- Three serial-number conventions in real-world use, and that the only
  rule BIND actually enforces is "always numerically greater than
  before"
- The complete, formal rule for absolute (trailing dot) vs relative (no
  trailing dot) names — and a demonstrated, real example of how getting
  this wrong produces a silently "valid" zone file with the wrong data
  in it, undetectable by `named-checkzone`

## Commands to Remember

```bash
sudo named-checkzone internal.example.com /var/named/internal.example.com.zone
dig @192.168.10.10 web01.internal.example.com
dig @192.168.10.10 internal.example.com SOA
sudo systemctl reload named
journalctl -u named -n 20 --no-pager
```

(No new commands this module — the focus is entirely on reading and
writing zone-file content correctly; the verification toolchain is the
same as Module 07.)

## Practical Lab

1. Add an explicit per-record TTL override to one record in your zone
   file (e.g. set `web01`'s TTL to `300`), reload, and confirm with
   `dig` that the returned TTL differs from the zone's `$TTL` default
   for that one record specifically.
2. Deliberately reproduce the missing-trailing-dot mistake from section
   6 on a **test record** you don't otherwise need (not `web01` — pick
   a throwaway name), confirm `named-checkzone` reports no error, then
   confirm with `dig` that the record you intended doesn't actually
   exist. Remove the test record afterward.
3. Add a second A record for an existing host, giving it two IPs
   temporarily (e.g. add a second `web01 IN A 192.168.10.21` line),
   reload, and observe `dig` return both addresses in the ANSWER
   section. Remove the extra line afterward to keep the zone matching
   the rest of the course.
4. Write out, from memory, the difference between what a blank name
   field means versus what a bare `@` means versus what a trailing dot
   means.

## Troubleshooting Exercise

**Scenario:** a teammate added a new host record to the zone file,
bumped nothing else, saved, and reloaded. `named-checkzone` reports
`OK`, `named` reloads without error, but the *old* data is still being
returned by `dig` from a *different* machine on the network (not DNS01
itself).

1. **Symptom:** DNS01 itself (`dig @192.168.10.10 ...` run locally on
   DNS01) shows the new record correctly, but a remote machine querying
   the same name still sees old data.
2. **Diagnosis:** this is very likely **not** a zone-file problem at
   all — DNS01 is serving the new data correctly. The remaining
   suspect is caching somewhere in the path between the remote machine
   and DNS01.
   ```bash
   dig @192.168.10.10 <name>              # confirm DNS01 itself is correct
   dig <name>                              # from the remote machine, using its default resolver
   dig @192.168.10.10 <name>               # from the remote machine, targeting DNS01 directly
   ```
3. **Commands:** compare the three `dig` results above; check the TTL
   value on the record in question.
4. **Root cause:** almost certainly the remote machine's own resolver
   (or an intermediate caching resolver) is still serving a cached
   answer from before the change, and the record's TTL simply hasn't
   expired yet — this is expected, correct DNS behavior, not a bug.
5. **Fix:** none needed if this is simply TTL-driven propagation delay
   — wait out the TTL. If the team needs faster propagation for future
   changes, that's exactly the "lower the TTL in advance of a planned
   change" pattern from section 2.
6. **Verification:** re-query after the original TTL window has
   elapsed and confirm the remote machine now sees the updated value.

## Interview Questions

- **What's the difference between `$TTL` and a per-record TTL value?**
  Short answer: `$TTL` sets the file-wide default; a value explicitly
  placed on an individual record overrides that default for that
  record only. Detailed: this lets a zone administrator apply a
  uniform caching policy to most records while selectively shortening
  (or lengthening) it for specific records that need different
  propagation behavior. Real-world example: lowering one record's TTL
  days ahead of a planned server migration so that, when the actual
  cutover happens, clients worldwide pick up the new IP quickly instead
  of continuing to hit the decommissioned server for up to the zone's
  full default TTL.

- **Why can't a name with a CNAME record have any other record type?**
  Short answer: it's a hard DNS protocol rule — a CNAME says "for
  every question about this name, go ask about this other name
  instead," which is fundamentally incompatible with also directly
  answering some other question type at the same name. Detailed: this
  is why, for example, a domain's bare apex/root name (`example.com`
  itself) generally cannot be a CNAME if it also needs an MX record (as
  almost every real domain does) — this exact limitation is one reason
  some DNS providers offer non-standard "ALIAS" or "ANAME" record types
  as CNAME-like workarounds specifically for zone apexes. Real-world
  example: a team wanting `example.com` (not `www.example.com`) to
  point at a CDN that only issues CNAME targets runs directly into this
  restriction, and has to use either that provider's proprietary
  workaround record type or a plain A record with an IP instead.

- **Why is "always numerically greater" the only rule for serial
  numbers, and why does that make the specific convention you choose
  mostly a team-preference decision?**
  Short answer: BIND's actual comparison logic doesn't care about
  format — it purely compares magnitude — so `YYYYMMDDNN`, a plain
  incrementing integer, and a Unix timestamp are all equally valid as
  long as each new value is larger than the last. Detailed: what
  actually causes real production incidents isn't the choice of
  convention, but a human forgetting to bump the serial at all, or
  (much worse) restoring an old zone file/backup whose serial is lower
  than what secondaries already have — the latter is precisely why
  Module 27's backup/restore procedure has to account for serial
  numbers explicitly, not just file content. Real-world example: a team
  standardizing on `YYYYMMDDNN` specifically because it lets anyone
  glance at a zone file's SOA line and immediately know roughly when it
  was last changed, without needing to check version control history
  or ask around.

## Expected Result

You can now read any zone file and correctly determine, for every
single name in it, whether it's absolute or relative and what it
actually resolves to — including catching the exact class of "valid
syntax, wrong data" mistake that `named-checkzone` cannot detect for
you. You understand `$TTL` and `$ORIGIN` as the two directives
governing an entire file's default behavior, and you have a working,
correctly serial-numbered habit for every future zone-file edit in this
course.

`internal.example.com`'s **forward** zone (name → IP) is now
thoroughly understood. **MODULE 09 — Reverse DNS** builds the mirror
image — IP → name — for this exact same subnet, introducing
`in-addr.arpa` and PTR records for real.

---

Say **NEXT** to continue to Module 09 — Reverse DNS.
