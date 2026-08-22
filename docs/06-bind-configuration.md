# MODULE 06 — Basic BIND Configuration

> Course: BIND DNS from Zero | Distro: AlmaLinux / Rocky / RHEL
> Builds on: Module 05 (`named` installed and running with default config)
> Edits: `/etc/named.conf` on **DNS01** — `192.168.10.10`

`named` is running, but right now it's serving the distro's default,
generic configuration — not answering for your network, not reachable
from anywhere but `127.0.0.1`. This module opens `/etc/named.conf` for
the first time, explains every directive you'll actually touch in this
course, and — critically — introduces `named-checkconf`, the safety net
you must use before every single reload from this point forward.

---

## 1. What this module covers

- The structure of `/etc/named.conf`: statements, blocks, `{}`, `;`
- The `options {}` block and its most important directives:
  `directory`, `listen-on`, `listen-on-v6`, `allow-query`, `recursion`,
  `forwarders`, `dnssec-validation`
- A first look at `logging {}` and `zone {}` (both covered in full
  depth in later modules — Module 20 and Module 07/08 respectively)
- Building a minimal, working configuration for DNS01
- `named-checkconf` — validating syntax before you ever reload
- What happens, concretely, when configuration syntax is wrong

## 2. The structure of `named.conf`

BIND's configuration language is its own small, C-like syntax:

```text
statement-name {
    setting value;
    nested-block {
        another-setting value;
    };
};
```

**Rules to internalize now, because a single mistake here is the #1
cause of "reload silently did nothing" confusion for beginners:**

- Every setting line ends in a **semicolon** `;` — including the
  closing `};` of nested blocks
- Blocks are wrapped in curly braces `{ }`
- Comments use `//`, `#`, or C-style `/* ... */`
- Whitespace/indentation is cosmetic only — BIND doesn't care about it,
  but consistent indentation is what makes a `named.conf` file humanly
  reviewable, which matters enormously once you have several zones
  declared

**View the default file:**
```bash
cat /etc/named.conf
```

**Default RHEL-family content (abbreviated, comments trimmed):**
```text
options {
        listen-on port 53 { 127.0.0.1; };
        listen-on-v6 port 53 { ::1; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        recursing-file  "/var/named/data/named.recursing";
        secroots-file   "/var/named/data/named.secroots";
        allow-query     { localhost; };

        recursion yes;

        dnssec-validation yes;

        managed-keys-directory "/var/named/dynamic";
        pid-file "/run/named/named.pid";
        session-keyfile "/run/named/session.key";
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

zone "." IN {
        type hint;
        file "named.ca";
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
```

This explains exactly why Module 05's `ss -lntup` check showed BIND
listening only on `127.0.0.1` — `listen-on port 53 { 127.0.0.1; };` and
`allow-query { localhost; };` are both explicitly restricting it to the
local machine only. We fix both of these deliberately in this module.

---

## 3. The `options {}` block — directive by directive

### `directory`
```text
directory "/var/named";
```
Sets the **base path** BIND resolves all relative zone-file paths
against. When Module 07 declares `file "internal.example.com.zone";`
inside a `zone {}` block, BIND looks for it at
`/var/named/internal.example.com.zone` because of this setting.

### `listen-on port 53 { ... };`
```text
listen-on port 53 { 127.0.0.1; 192.168.10.10; };
```
**What it controls:** which IPv4 addresses (on which local interfaces)
`named` actually binds to and listens on. The default `{ 127.0.0.1; }`
means only processes on the same machine can reach it at all — no
other host on the network can even attempt a connection, regardless of
firewall or `allow-query` settings, because the socket itself isn't
bound to any externally-reachable address.

**We change this in this module** to include DNS01's real static IP,
so CLIENT01 (Module 22) and other lab hosts can actually reach it.

A common shorthand you'll see in real-world configs:
```text
listen-on port 53 { any; };
```
`any` binds to every IPv4 address on the machine — convenient, but less
explicit than listing addresses individually. This course prefers being
explicit (listing `127.0.0.1` and the static IP by name) because it's
more auditable and matches good production practice — you should be
able to read `named.conf` and know exactly which addresses are
intentional.

### `listen-on-v6`
Same concept, for IPv6. We'll set this explicitly too, even though our
lab is IPv4-focused, so the setting is never ambiguous:
```text
listen-on-v6 port 53 { none; };
```
`none` explicitly disables IPv6 listening — a deliberate, documented
choice, rather than leaving IPv6 behavior to distro defaults.

### `allow-query`
```text
allow-query { localhost; internal-networks; };
```
**What it controls:** which **clients** (by source IP/network) are
permitted to send this server queries at all — this is distinct from
`listen-on`, which controls what the server binds to. Think of
`listen-on` as "which doors exist" and `allow-query` as "who's allowed
to knock." Both must permit a client for a query to succeed.

We'll define an **ACL** (access control list, a named, reusable group
of addresses) for our lab subnet rather than hardcoding the raw CIDR
inline every time — full ACL depth is Module 17, but we introduce the
basic pattern here because it's needed for a working config immediately:

```text
acl internal-networks {
    192.168.10.0/24;
};
```

### `recursion`
```text
recursion yes;
```
**What it controls:** whether this server will do the full recursive
walk-the-hierarchy work (Module 03/11) on behalf of clients, versus
only answering authoritatively for zones it directly hosts. `yes` is
correct for DNS01's role as the internal recursive resolver for
CLIENT01 — but combined with `allow-query`/`allow-recursion` scoping,
**not** left wide open to the entire internet (an "open resolver,"
Module 11/17's central security warning).

### `forwarders`
```text
forwarders { 8.8.8.8; 1.1.1.1; };
```
**What it controls:** when DNS01 needs to resolve something it isn't
authoritative for and doesn't have cached, should it do the full
root→TLD→authoritative walk itself, or hand that work off to another
specific resolver? Full depth in Module 12 — we'll add this directive
there. For this module's minimal config, we introduce it but you can
leave it commented out if you'd rather DNS01 do full recursion itself
for now; both are valid choices, revisited properly in Module 12.

### `dnssec-validation`
```text
dnssec-validation yes;
```
**What it controls:** whether `named`, acting as a recursive resolver,
cryptographically validates DNSSEC signatures on answers it receives
from other (DNSSEC-signed) authoritative servers, rejecting answers
that fail validation. `yes` (the modern default) is the correct,
secure choice — leave it as-is. DNSSEC *signing your own zone* is a
separate, more advanced topic not covered hands-on in this course, but
validating *other* zones' DNSSEC signatures as a recursive resolver is
already happening by default the moment this is `yes`.

---

## 4. A minimal working configuration for DNS01

Here is the actual, complete `options {}` block we'll build toward in
this module — this exact block is also saved in this repository at
[`configs/primary/named.conf.options`](../configs/primary/named.conf.options)
for reference:

```text
acl internal-networks {
    192.168.10.0/24;
};

options {
    directory "/var/named";

    listen-on port 53 { 127.0.0.1; 192.168.10.10; };
    listen-on-v6 port 53 { none; };

    allow-query { internal-networks; localhost; };
    allow-recursion { internal-networks; localhost; };

    recursion yes;

    dnssec-validation yes;

    pid-file "/run/named/named.pid";
    session-keyfile "/run/named/session.key";
};

logging {
    channel default_debug {
        file "data/named.run";
        severity dynamic;
    };
};

zone "." IN {
    type hint;
    file "named.ca";
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
```

**What changed from the distro default, and why, line by line:**
- Added the `internal-networks` ACL — a named, reusable definition of
  "our lab subnet," used in two places below instead of repeating the
  raw CIDR
- `listen-on` now includes `192.168.10.10` — DNS01 is reachable from
  the network, not just itself
- `allow-query` now includes `internal-networks` — lab hosts are
  permitted to query it (note: `localhost` is kept too, deliberately —
  DNS01 itself, and tools run locally on it, should still work)
- Added `allow-recursion { internal-networks; localhost; };`
  explicitly — this is a **second, distinct** ACL check specifically
  for recursive queries, separate from `allow-query`. This distinction
  matters a great deal for security (Module 17): you can have a zone be
  publicly *queryable* (for authoritative answers about that zone)
  while keeping *recursion* restricted to trusted networks only —
  exactly the split that prevents your server from becoming an open
  resolver.
- `zone "." { type hint; ... };` and the two `include` lines are kept
  unchanged from the default — these bring in the root hints (Module
  03's root server bootstrap list) and RHEL's own pre-built localhost/
  reverse-localhost zone definitions, which are safe and standard to
  keep as-is

**Edit the live file directly:**
```bash
sudo cp /etc/named.conf /etc/named.conf.orig
sudo vi /etc/named.conf
```
Copying a `.orig` backup before your first hand-edit is good practice —
you'll want an easy rollback reference the first few times you're
learning this syntax.

---

## 5. `named-checkconf` — validate before you ever reload

This is the single most important habit this module builds. **Never
reload `named` after a config edit without running this first.**

```bash
sudo named-checkconf
```

**What it does:** parses `/etc/named.conf` (and everything it
`include`s) purely for **syntax** correctness — missing semicolons,
unbalanced braces, unknown directive names, malformed ACLs — without
touching the running service at all.

**Expected output on success:** **nothing** — genuinely no output at
all is success. This surprises almost everyone the first time; silence
means "no errors found."

**Verify your understanding by deliberately breaking it:** remove a
semicolon from the file (e.g. after `directory "/var/named"`) and run
`named-checkconf` again:

```bash
sudo named-checkconf
```

**Expected output on failure:**
```
/etc/named.conf:8: missing ';' before 'listen-on'
```
This tells you the **exact file and line number** of the problem — put
the semicolon back, save, and confirm `named-checkconf` goes silent
again before moving on.

**A more thorough variant, checking zone files too (previewed here,
used for real starting Module 07):**
```bash
sudo named-checkconf -z
```
`-z` additionally attempts to load every zone declared in the config
(not just check `named.conf`'s own syntax) — this will become relevant
the moment you declare your first real zone in the next module.

---

## 6. Applying the configuration

Once `named-checkconf` is silent (success):

```bash
sudo systemctl restart named
```

**Why `restart` and not `reload` here specifically:** this is exactly
the case flagged in Module 05 — you changed core `options {}` settings
(`listen-on`, `allow-query`, `allow-recursion`), which are established
once at process start and are **not** safely re-appliable via `reload`.
`reload` is for zone-file content changes (Module 07 onward); core
listener/ACL changes in `options {}` need a full `restart`.

**Verify:**
```bash
systemctl status named
sudo ss -lntup | grep ':53'
```
**Expected `ss` output now shows the static IP, not just loopback:**
```
udp   UNCONN 0      0      192.168.10.10:53   0.0.0.0:*   users:(("named",...))
udp   UNCONN 0      0      127.0.0.1:53       0.0.0.0:*   users:(("named",...))
tcp   LISTEN 0      10     192.168.10.10:53   0.0.0.0:*   users:(("named",...))
tcp   LISTEN 0      10     127.0.0.1:53       0.0.0.0:*   users:(("named",...))
```

```bash
dig @192.168.10.10 . NS
```
This should now succeed — you're querying DNS01 by its real network
address for the first time, rather than only via `127.0.0.1`, and
getting a real answer back from the root hints, confirming both
`listen-on` and `allow-query` are correctly permitting this.

**Note:** the firewall (`firewalld`, still unconfigured for port 53 at
this point — that's Module 19) may still block this from a *different*
machine, even though it now works from DNS01 itself. Don't be alarmed
if `dig @192.168.10.10 ...` run from CLIENT01 doesn't work yet — that's
expected until Module 19; running it from DNS01 itself against its own
static IP is the correct verification for this module.

---

## 7. What happens when configuration syntax is wrong, end to end

Walk through this deliberately once, so the failure mode is familiar
rather than alarming the first time you hit it for real.

**Step 1 — introduce a real syntax error** (missing closing brace):
```bash
sudo vi /etc/named.conf
# delete a single closing `};` somewhere in the acl or options block
```

**Step 2 — check first, as always:**
```bash
sudo named-checkconf
```
```
/etc/named.conf:15: unexpected end of file
```

**Step 3 — what happens if you skip checking and reload/restart
anyway:**
```bash
sudo systemctl restart named
systemctl status named
```
**Expected output:**
```
● named.service - Berkeley Internet Name Domain (DNS)
     Loaded: loaded (/usr/lib/systemd/system/named.service; enabled; preset: disabled)
     Active: failed (Result: exit-code) since ...
```
`failed`, not `active (running)` — `named` refused to start at all with
invalid config, rather than starting with a partial/wrong
configuration. This fail-closed behavior is a deliberate, good safety
property of BIND: it does not silently ignore config errors and run
with whatever it could partially parse.

**Step 4 — find the specific error:**
```bash
journalctl -u named -n 20 --no-pager
```
**Expected output includes a line like:**
```
named[6102]: /etc/named.conf:15: unexpected end of file
named[6102]: loading configuration: failure
named[6102]: exiting (due to fatal error)
```
Note this is the **same error message** `named-checkconf` already gave
you — which is exactly why you check first: `named-checkconf` gives you
the identical diagnostic information in about a tenth of a second,
without any service downtime, before you ever touch the live service.

**Step 5 — fix and restore service:**
```bash
sudo vi /etc/named.conf   # restore the missing brace
sudo named-checkconf      # confirm silent/clean
sudo systemctl restart named
systemctl status named    # confirm active (running) again
```

---

## What I Learned

- The syntax rules of `named.conf`: statements, `{}` blocks, mandatory
  trailing `;`, comment styles
- What each core `options {}` directive controls:
  `directory`, `listen-on`/`listen-on-v6`, `allow-query`, `recursion`,
  `allow-recursion`, `forwarders`, `dnssec-validation`
- The distinction between `listen-on` ("which doors exist") and
  `allow-query`/`allow-recursion` ("who's allowed to knock, and for
  what") — two independent layers that both must permit a client
- How to build a named, reusable ACL (`acl internal-networks { ... };`)
  instead of repeating raw CIDR blocks
- That `named-checkconf` must be run before every reload/restart, and
  that silent output means success
- That `restart` is required (not `reload`) for `options {}`-level
  changes like `listen-on` or ACLs
- That BIND fails closed on invalid config — `named` will not start at
  all rather than run with a broken partial configuration

## Commands to Remember

```bash
cat /etc/named.conf
sudo cp /etc/named.conf /etc/named.conf.orig
sudo vi /etc/named.conf
sudo named-checkconf
sudo named-checkconf -z
sudo systemctl restart named
systemctl status named
journalctl -u named -n 20 --no-pager
sudo ss -lntup | grep ':53'
dig @192.168.10.10 . NS
```

## Practical Lab

1. Back up `/etc/named.conf`, then edit it to match this module's
   minimal configuration (ACL, `listen-on`, `allow-query`,
   `allow-recursion`, `recursion`, `dnssec-validation`).
2. Run `named-checkconf` and confirm silent success before doing
   anything else.
3. `restart` `named` and confirm `active (running)` via
   `systemctl status`.
4. Confirm with `ss -lntup` that DNS01 is now listening on
   `192.168.10.10:53` (both UDP and TCP), not just `127.0.0.1`.
5. Run `dig @192.168.10.10 . NS` from DNS01 itself and confirm a
   successful answer.
6. Deliberately remove one semicolon, run `named-checkconf`, read the
   exact line-number error it gives you, then fix it and confirm clean
   output again.
7. Deliberately remove one closing brace, skip `named-checkconf`,
   `restart` anyway, observe `systemctl status named` reporting
   `failed`, find the same error in `journalctl -u named`, then fix and
   restore service.

## Troubleshooting Exercise

**Scenario:** after editing `named.conf` to add the ACL and new
`allow-query` block, a teammate runs `dig @192.168.10.10 . NS` from
DNS01 and gets `connection refused` instead of an answer or even a
`REFUSED` DNS-level response.

1. **Symptom:** `dig` reports a *connection*-level failure (no DNS
   response at all — different from a DNS-level `REFUSED` status,
   which would still mean the server answered, just declined).
2. **Diagnosis:**
   ```bash
   systemctl status named
   sudo ss -lntup | grep ':53'
   ```
3. **Commands:** `systemctl status named`, `journalctl -u named -n 20`,
   `named-checkconf`
4. **Root cause (most likely, given the symptom):** `named` isn't
   running at all — most commonly because the edited config had a
   syntax error and `restart` failed silently from the operator's
   perspective (they didn't check `systemctl status` immediately
   after).
5. **Fix:**
   ```bash
   sudo named-checkconf
   # fix whatever it reports
   sudo systemctl restart named
   ```
6. **Verification:** `systemctl status named` shows `active (running)`;
   `dig @192.168.10.10 . NS` succeeds.

## Interview Questions

- **What's the difference between `listen-on` and `allow-query` in
  `named.conf`?**
  Short answer: `listen-on` controls which local addresses `named`
  binds to at all; `allow-query` controls which client source
  addresses are permitted to query it once it's listening. Detailed:
  they're independent layers — a client can be on an address `named`
  is listening on but still be denied by `allow-query`, and conversely
  a client on an `allow-query`-permitted network still can't connect if
  `named` never bound to a reachable address in the first place.
  Real-world example: a common, correct production pattern is
  `listen-on { any; }` (bind everywhere) combined with a restrictive
  `allow-query` ACL (only permit specific trusted networks) — binding
  broadly but restricting who's actually allowed to ask.

- **Why does BIND distinguish `allow-query` from `allow-recursion` as
  two separate ACLs?**
  Short answer: `allow-query` governs who can ask this server anything
  at all (including authoritative answers for zones it hosts);
  `allow-recursion` specifically governs who can make it perform
  recursive resolution on their behalf. Detailed: this split lets a
  server be authoritative-and-public for its own zones (anyone can ask
  "what's the A record for my domain") while keeping expensive/
  abusable recursive resolution restricted to trusted internal clients
  only — directly preventing the open-resolver misconfiguration covered
  in Module 11/17. Real-world example: a company's public-facing
  authoritative DNS server for `example.com` should have wide-open
  `allow-query` (the whole internet needs to resolve their website) but
  `allow-recursion { none; };` — it should never do recursive lookups
  on behalf of internet strangers.

- **Why run `named-checkconf` before every reload/restart instead of
  just trying the reload and seeing if it fails?**
  Short answer: `named-checkconf` gives you the exact same syntax
  diagnostic instantly, with zero service impact, before you ever touch
  the running service. Detailed: because BIND fails closed on invalid
  config (Module 06's demonstrated `failed` state), skipping validation
  risks a live service outage — however brief — purely from a typo,
  which is entirely avoidable. Real-world example: in a production
  change-management or CI/CD pipeline (foreshadowing Module 25), running
  `named-checkconf` (and `named-checkzone` for each zone) as an
  automated pre-deploy gate is standard practice specifically to
  guarantee a bad config can never even reach the point of being
  applied to a live server.

## Expected Result

DNS01 now runs a deliberately-written, understood configuration: it
listens on its real network address, restricts both query and
recursion access to an explicit internal-networks ACL, validates
DNSSEC on upstream answers, and you've verified — both by breaking it
on purpose and by fixing it — exactly how `named-checkconf` catches
problems before they ever reach the running service.

DNS01 still isn't authoritative for anything of your own yet — it's
only serving root hints and the distro's default localhost zone. That
changes in **MODULE 07 — Primary Authoritative DNS**, where you declare
and build your first real zone, `internal.example.com`, from scratch.

---

Say **NEXT** to continue to Module 07 — Primary Authoritative DNS.
