# MODULE 05 — Install BIND

> Course: BIND DNS from Zero | Distro: AlmaLinux / Rocky / RHEL
> Builds on: Modules 01–04 (server ready, networking + DNS concepts, `dig` fluency)
> Installs onto: **DNS01** — `192.168.10.10`

This is the module where DNS01 stops being "a Linux server that could
one day run DNS" and becomes an actual DNS server. Everything before
this was deliberately preparation — from here forward, every module
builds directly on `named` actually running.

---

## 1. What this module covers

- What `bind` and `bind-utils` actually are, and how they differ
- Installing BIND on AlmaLinux/Rocky/RHEL
- The `named` service — start, stop, restart, reload, enable
- The critical difference between `restart` and `reload`
- Every important filesystem path BIND uses, and what lives where
- File ownership and permissions, and why they matter for BIND
  specifically

## 2. What `bind` actually is — package vs daemon vs protocol

Three names get used almost interchangeably in casual conversation, but
they refer to three different things:

- **BIND** (Berkeley Internet Name Domain) — the *software project*,
  developed and maintained by ISC (Internet Systems Consortium). It's
  one implementation of the DNS protocol among several (others include
  PowerDNS, Knot DNS, Unbound) — but it's the most widely deployed
  authoritative/recursive DNS server software in the world, and the one
  this entire course is built around.
- **`named`** — the actual **daemon** (background service process) that
  BIND ships. When you `systemctl start named`, this is the process
  that comes up and starts listening on port 53. "BIND" and "named" are
  often used interchangeably in conversation, but technically BIND is
  the software package/project, `named` is the specific running process.
- **DNS** — the protocol itself (Module 03), which `named` is one
  possible implementation of.

**Why this distinction is worth knowing explicitly:** in interviews and
in production incident discussions, precision here signals real
understanding — "BIND is down" is casual shorthand for "the `named`
process/service is down," and being able to state that precisely
matters when you're writing an incident report or explaining an issue
to a team that might run a different DNS server implementation
elsewhere in the same infrastructure.

---

## 3. Installing BIND

```bash
sudo dnf install -y bind bind-utils
```

**What each package provides:**

| Package | Provides |
|---|---|
| `bind` | The `named` daemon itself, default config files, systemd unit, root hints file |
| `bind-utils` | Client tools: `dig`, `host`, `nslookup`, `named-checkconf`, `named-checkzone` |

You already installed `bind-utils` in Module 04 — running the command
again with both packages listed is safe; `dnf` will simply confirm
`bind-utils` is already installed and only install `bind` as new.

**Expected output (abbreviated):**
```
Installing:
 bind                    x86_64   32:9.16.23-...   appstream   ...
Installing dependencies:
 bind-libs                x86_64   32:9.16.23-...   appstream   ...
 bind-license              noarch   32:9.16.23-...   appstream   ...
 python3-bind              noarch   32:9.16.23-...   appstream   ...

Complete!
```

**Ubuntu/Debian note:** the package name differs —
```bash
sudo apt install -y bind9 bind9-utils bind9-dnsutils
```
`bind9` is the daemon, `bind9-utils` provides `named-checkconf` /
`named-checkzone`, and `bind9-dnsutils` provides `dig`/`host`/
`nslookup`. The service unit is `named` on RHEL-family but `bind9` on
Debian/Ubuntu — a detail that will matter every time you type
`systemctl` commands going forward if you're following along on Ubuntu.

---

## 4. The `named` systemd service — full lifecycle

```bash
systemctl status named
```

**Expected output immediately after install (not yet started):**
```
○ named.service - Berkeley Internet Name Domain (DNS)
     Loaded: loaded (/usr/lib/systemd/system/named.service; disabled; preset: disabled)
     Active: inactive (dead)
```

Note `disabled` (won't auto-start on boot) and `inactive (dead)` (not
currently running) — this is the expected, correct state right after a
fresh install, before you've configured or started anything.

### Enable and start

```bash
sudo systemctl enable named
sudo systemctl start named
```

Or, in one step:
```bash
sudo systemctl enable --now named
```
**What `--now` does:** combines "enable at boot" and "start
immediately" into a single command — you'll use this pattern for every
service you bring up for the rest of this course.

### Verify it actually started

```bash
systemctl status named
```

**Expected output:**
```
● named.service - Berkeley Internet Name Domain (DNS)
     Loaded: loaded (/usr/lib/systemd/system/named.service; enabled; preset: disabled)
     Active: active (running) since Sat 2026-08-22 15:32:04 IST; 4s ago
       Docs: man:named(8)
   Main PID: 5821 (named)
      Tasks: 5 (limit: 10998)
     Memory: 22.1M
        CPU: 65ms
     CGroup: /system.slice/named.service
             └─5821 /usr/sbin/named -u named -c /etc/named.conf
```

Confirm with the tools from Module 02:
```bash
sudo ss -lntup | grep ':53'
```
**Expected output:**
```
udp   UNCONN 0      0            192.168.10.10:53    0.0.0.0:*    users:(("named",pid=5821,fd=20))
tcp   LISTEN 0      10           192.168.10.10:53    0.0.0.0:*    users:(("named",pid=5821,fd=21))
```

This is the exact UDP+TCP dual-listener pattern predicted in Module 02
— now real, on your own server, running the default configuration
straight out of the box (which, as shipped on RHEL-family, only
listens on `127.0.0.1` by default in some distro builds — if your
output shows `127.0.0.1:53` instead of your static IP, that's expected
until Module 06 explicitly configures `listen-on`; don't troubleshoot
that yet).

And confirm with Module 04's `dig`:
```bash
dig @127.0.0.1 . NS
```
Querying `.` (the root zone) for its `NS` records against your own
`named` process, using its default out-of-the-box configuration, is a
standard smoke test — a healthy fresh install with root hints loaded
correctly should return an answer (from cache/hints) rather than a
connection failure. A connection failure here means `named` isn't
actually listening/reachable at all, and is your very first real
troubleshooting checkpoint — go back to `systemctl status named` and
`journalctl -u named` before proceeding.

---

## 5. `restart` vs `reload` — a distinction that matters constantly from here on

```bash
sudo systemctl restart named
sudo systemctl reload named
sudo systemctl stop named
```

**`restart`** — fully stops the `named` process and starts a brand new
one. This:
- Re-reads **all** configuration from scratch (`named.conf` and every
  zone file)
- Drops the **entire cache** (any recursive/cached answers are lost —
  Module 13's cache is gone, cold-start again)
- Causes a brief window (typically well under a second, but non-zero)
  where port 53 isn't being served at all
- Is the heavier-handed option — use it when you've changed core
  `named.conf` `options {}` settings that reload can't safely pick up

**`reload`** — signals the *already-running* `named` process to re-read
its configuration and zone files **without restarting the process
itself**. This:
- Is much faster and has effectively no service interruption
- **Preserves the existing cache** — recursive/cached answers survive
- Is what you use after editing a zone file (Module 07 onward) — this
  will become your most frequently run command in this entire course
- Internally, this is equivalent to running `rndc reload` (`rndc` is
  BIND's dedicated remote-control utility, briefly introduced here and
  used more in Module 20/21)

**Production framing:** in a real environment, `reload` is strongly
preferred whenever it's sufficient, precisely because it avoids
dropping the cache and avoids any service interruption — `restart`
should be treated as the "when reload genuinely isn't enough" option,
not the default reflex.

**`stop`** — shuts the process down entirely; port 53 is no longer
served at all until you `start` it again.

**Verify a reload actually happened (rather than silently failing):**
```bash
sudo systemctl reload named
journalctl -u named -n 20 --no-pager
```
Look for a log line confirming the reload completed (e.g. "all zones
loaded" or similar, exact wording varies by BIND version) — **a reload
that fails due to a config syntax error does not necessarily produce an
obvious error on the terminal**, which is exactly why Module 06
introduces `named-checkconf` as a **mandatory** pre-reload validation
step, not an optional nicety.

---

## 6. Important filesystem paths

BIND on RHEL-family systems spreads its files across several
directories with a specific, security-conscious layout. Learn these
now — you'll reference every one of them repeatedly for the rest of the
course.

```bash
ls -la /etc/named.conf
ls -la /etc/named/
ls -la /var/named/
ls -la /run/named/
```

| Path | What lives here |
|---|---|
| `/etc/named.conf` | The **main** configuration file — global `options {}`, `zone {}` declarations, logging, ACLs. Everything in Module 06 onward edits this file directly, or files it includes. |
| `/etc/named/` | On some RHEL-family layouts, holds included config fragments (e.g. `/etc/named/named.conf.local`-style splits) — not always used; many setups keep everything in `/etc/named.conf` directly, which is what this course does for clarity. |
| `/var/named/` | **Zone files** live here — the actual DNS record data. The default install includes some pre-made files here for the root zone hints and localhost zones — inspect them now: `ls -la /var/named/` |
| `/var/named/data/` | Runtime data BIND writes itself, e.g. `named.run` (a plain-text log of the most recent run), cache statistics dumps |
| `/var/named/dynamic/` | Used for **dynamically updated** zone data (DNSSEC key material if enabled, DDNS-updated records) — not heavily used in this course's static zone-file approach, but SELinux (Module 18) specifically cares about this directory having correct context |
| `/var/log/` (specific files configured in Module 20) | Where BIND's own query/security/general logs land, once you configure logging channels explicitly |
| `/run/named/` | Runtime files — notably `named.pid` (the running process's PID file) and the control socket used by `rndc` |

**Inspect the default zone files now:**
```bash
cat /var/named/named.ca
```
**What this is:** the **root hints file** — a static list of the 13
root server names and IP addresses (Module 03). This is how `named`
knows where to start an iterative walk before it's ever resolved
anything — it doesn't discover root servers via DNS itself (that would
be circular), it ships with this file baked in. ISC updates this file
periodically as root server addresses change; on a real production
system, keeping this file current matters, though it changes rarely.

```bash
cat /var/named/named.localhost
```
**What this is:** the default zone file for `localhost` — a small,
pre-built example of exactly the zone-file syntax you'll be writing
from scratch in Module 07. Skim it now; it will look far less
mysterious once you've completed Module 08's deep dive, but it's worth
noticing the shape of it already:
```
$TTL 1D
@       IN SOA  @ rname.invalid. (
                                        0       ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
        NS      @
        A       127.0.0.1
        AAAA    ::1
```

---

## 7. File ownership and permissions

```bash
ls -l /etc/named.conf
ls -l /var/named/
```

**Expected output:**
```
-rw-r-----. 1 root named 1234 Aug 22 15:20 /etc/named.conf
drwxrwx---+ 7 root named   61 Aug 22 15:20 /var/named
```

**Why this matters:** the `named` daemon does **not** run as `root`
(confirm this yourself: `ps aux | grep named` shows the process owned
by the `named` user — you can also see this directly in the
`systemctl status named` output above: `/usr/sbin/named -u named ...`,
where `-u named` explicitly tells `named` to drop privileges to the
`named` user after binding to port 53).

This means:
- `/etc/named.conf` and everything under `/var/named/` must be readable
  (and, for `/var/named/dynamic/`, writable) by the **`named` group**,
  not just `root`
- If you ever create a new zone file by hand and it ends up owned by
  `root:root` with mode `600`, `named` will fail to read it — a very
  common, very confusing early mistake, because the *syntax* of your
  zone file might be perfect and it will still fail to load, purely on
  a permissions error
- **Never "fix" this by making files world-readable/writable as a
  shortcut** — the correct fix is always matching group ownership
  (`named`) and reasonable permissions (typically `640` for files,
  `750` for directories), exactly matching what the default install
  already demonstrates above

**The command you'll reach for whenever you create a new zone file by
hand (previewed here, used for real starting Module 07):**
```bash
sudo chown root:named /var/named/your-new-zone.db
sudo chmod 640 /var/named/your-new-zone.db
```

**Why `named` binding to port 53 (< 1024) without running as root
doesn't violate the "privileged ports need root" rule you may already
know:** `named` briefly runs as root only long enough to bind the
privileged port, then immediately drops to the unprivileged `named`
user for everything else (this is a standard, well-understood Unix
daemon security pattern, not unique to BIND) — worth knowing precisely
for Module 17/18's security discussion.

---

## What I Learned

- The precise distinction between BIND (the software project), `named`
  (the running daemon), and DNS (the protocol)
- How to install BIND on RHEL-family (`bind` + `bind-utils`) vs Ubuntu
  (`bind9` + `bind9-utils` + `bind9-dnsutils`)
- The full `named` service lifecycle: `enable`, `start`, `stop`,
  `restart`, `reload` — and critically, why `reload` (preserves cache,
  no interruption) is preferred over `restart` (drops cache, brief
  interruption) whenever it's sufficient
- Every important BIND filesystem path and what lives in each:
  `/etc/named.conf`, `/var/named/`, `/var/named/data/`,
  `/var/named/dynamic/`, `/run/named/`
- Why `named` runs as the unprivileged `named` user (after briefly
  binding port 53 as root) and why file ownership/group permissions on
  zone files matter — and why loosening permissions is never the
  correct fix

## Commands to Remember

```bash
sudo dnf install -y bind bind-utils
sudo systemctl enable --now named
systemctl status named
sudo systemctl restart named
sudo systemctl reload named
sudo systemctl stop named
journalctl -u named -n 20 --no-pager
sudo ss -lntup | grep ':53'
dig @127.0.0.1 . NS
cat /var/named/named.ca
ls -l /etc/named.conf /var/named/
sudo chown root:named /var/named/<zonefile>
sudo chmod 640 /var/named/<zonefile>
```

## Practical Lab

1. Install `bind` and `bind-utils` on DNS01.
2. Enable and start `named`, and confirm `systemctl status named`
   shows `active (running)`.
3. Confirm port 53 is bound using `ss -lntup` — note whether it's bound
   to `127.0.0.1` or your static IP at this point (default config,
   before Module 06 changes it).
4. Run `dig @127.0.0.1 . NS` and confirm you get an answer back from
   the root hints.
5. Read through `/var/named/named.ca` and `/var/named/named.localhost`
   and identify the SOA line, NS line, and A line in the latter.
6. Run `ls -l /var/named/` and confirm every file is owned by
   `root:named`.
7. Practice the `restart` vs `reload` distinction: run
   `sudo systemctl reload named`, then check `journalctl -u named -n 10`
   to confirm the reload was logged.

## Troubleshooting Exercise

**Scenario:** you create a brand new zone file by hand later in Module
07 using a plain `sudo vi /var/named/newzone.db`, and after reloading,
`named` fails to load that zone.

1. **Symptom:** `dig` against that zone returns `SERVFAIL`, or
   `systemctl status named` / `journalctl -u named` shows an error
   mentioning "permission denied" for that specific file.
2. **Diagnosis:**
   ```bash
   ls -l /var/named/newzone.db
   ```
3. **Commands:** `ls -l`, `journalctl -u named -n 30`
4. **Root cause:** the file was created directly with `vi` as `root`,
   so it's owned `root:root` with restrictive default permissions —
   the `named` user (running as an unprivileged, dropped-privilege
   process) cannot read it.
5. **Fix:**
   ```bash
   sudo chown root:named /var/named/newzone.db
   sudo chmod 640 /var/named/newzone.db
   sudo systemctl reload named
   ```
6. **Verification:** re-run `dig` against a record in that zone and
   confirm `NOERROR` with a valid answer; confirm no permission errors
   appear in a fresh `journalctl -u named -n 10` after the reload.

## Interview Questions

- **What's the difference between `systemctl restart named` and
  `systemctl reload named`, and when would you use each?**
  Short answer: `restart` fully stops and starts the process (drops
  cache, brief service interruption); `reload` re-reads config/zones
  on the already-running process (cache preserved, no interruption).
  Detailed: `reload` is sufficient for zone file changes and most
  day-to-day content updates; `restart` is needed for certain core
  `options {}` changes that can't be safely applied to a running
  process. Real-world example: updating a single A record in a zone
  file and running `reload` keeps the recursive cache warm for
  thousands of other cached answers your resolver is serving to
  clients — an unnecessary `restart` for a one-line zone change would
  needlessly cold-start that cache and briefly interrupt service for no
  benefit.

- **Why does `named` run as a dedicated unprivileged user instead of
  root, and how does it still manage to bind to port 53 (a privileged
  port)?**
  Short answer: it briefly holds root privileges only long enough to
  bind port 53, then permanently drops to the unprivileged `named`
  user for everything else. Detailed: this "bind as root, then drop
  privileges" pattern is a standard Unix daemon security practice —
  limiting the blast radius if the running process is ever compromised,
  since an attacker exploiting `named` at runtime only gets the
  limited `named` user's privileges, not root. Real-world example: this
  is exactly why zone files must be owned/grouped so the `named` user
  can read them — the process genuinely does not have root's
  unrestricted file access after startup, which is a deliberate
  security boundary, not an oversight to work around.

- **Where does BIND store its zone data by default on RHEL-family
  systems, and why does that matter operationally?**
  Short answer: zone files live under `/var/named/`, separate from
  `/etc/named.conf`'s configuration. Detailed: this separation lets you
  apply different backup, permission, and change-management practices
  to configuration (rarely changes, security-sensitive) versus zone
  data (changes more often, is the actual content being served) —
  Module 27's backup strategy explicitly backs up both, but treats them
  as related-but-distinct concerns. Real-world example: in an
  automated deployment (Module 25), you might template `named.conf`
  once during provisioning but continuously deploy updated zone files
  under `/var/named/` as DNS records change — recognizing they're
  separate concerns with separate lifecycles.

## Expected Result

DNS01 now has `named` installed, enabled at boot, and running with the
distro's default out-of-the-box configuration. You've confirmed it's
listening on port 53 with `ss`, confirmed it answers a basic query with
`dig`, you understand every important filesystem path it uses, and you
know precisely when to `reload` versus `restart` it — a distinction
you'll apply in nearly every module from here forward.

DNS01 is not yet configured to serve *your* zones or accept queries
from other hosts on the network — that begins in **MODULE 06 — Basic
BIND Configuration**, where we edit `/etc/named.conf` for the first
time.

---

Say **NEXT** to continue to Module 06 — Basic BIND Configuration.
