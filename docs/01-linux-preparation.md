# MODULE 01 — Linux Server Preparation

> Course: BIND DNS from Zero | Distro: AlmaLinux / Rocky / RHEL (Ubuntu differences noted)
> Lab node built in this module: **DNS01** — `dns01.internal.example.com` — `192.168.10.10`

Before you can build a DNS server, you need a Linux server that is
correctly named, correctly addressed, and correctly reachable. Everything
you do for the rest of this course sits on top of what you do here. If
the hostname, `/etc/hosts`, or networking is wrong, DNS will misbehave in
ways that look like "DNS bugs" but are actually "Module 01 wasn't done
properly." So we start here, deliberately slowly.

---

## 1. What this module covers

By the end of this module your fresh VM/server will have:

- A real, permanent hostname (`dns01.internal.example.com`)
- A static IP address (`192.168.10.10/24`)
- A correct default gateway
- A working `/etc/hosts` file
- A known, working `/etc/resolv.conf`
- A firewall service you understand (even if not yet configured for DNS)
- SELinux in a known state (Enforcing — we do **not** disable it)
- Correct system time (NTP)
- Comfort with `systemctl` and `journalctl`, which you'll use in every
  future module to start/stop/inspect the `named` service

## 2. Why this matters (before you touch DNS)

DNS servers are extremely sensitive to identity and networking:

- BIND binds to a specific IP. If that IP changes later, your zone
  files, ACLs, and reverse DNS all break.
- Your **own hostname** frequently appears inside your own zone files
  (as the NS record target, for example). A wrong or unstable hostname
  means broken self-references.
- `/etc/resolv.conf` on the DNS server itself decides where *that
  machine* looks things up — this is different from what BIND serves to
  *others*. Confusing these two is one of the most common beginner
  mistakes in this whole course.

Get this right once, now, and you save yourself hours of confused
debugging in Modules 07, 09, and 21.

---

## 3. Installing the OS

This course assumes you provision a fresh minimal install of AlmaLinux 9
or Rocky Linux 9 (RHEL 9 works identically) — either as a VM (VirtualBox,
Proxmox, VMware) or a cloud instance. Use the **"Minimal Install"** or
**"Server"** base environment during installation; you don't need a
desktop environment for a DNS server.

During install, set:
- A root password or an admin user with `sudo` rights (either is fine —
  we'll use `sudo` throughout this course regardless of which you pick)
- Networking can be left as DHCP for now — we will convert it to static
  in this module

If you're on Ubuntu/Debian instead, install "Ubuntu Server" (no desktop).
Where Ubuntu diverges from RHEL-family in this module, it's called out
explicitly below.

---

## 4. First login — root vs normal user, and `sudo`

Log in either as `root` directly, or as your admin user and elevate with
`sudo`.

```bash
whoami
```

**What it does:** prints the currently logged-in user. Run this first —
if you expect to be `root` but see your own username, you're not
elevated yet, and commands later in this module (changing hostname,
editing network config) will fail with "Permission denied."

```bash
sudo whoami
```

**What it does:** `sudo` ("substitute user, do") temporarily runs the
following command as another user — by default, `root`. This confirms
your account is in the `wheel` group (RHEL-family) or `sudo` group
(Debian/Ubuntu) and can actually elevate.

**Why we don't just log in as root permanently:** in production,
logging in directly as root is avoided — it removes the audit trail of
*which admin* ran a command. Every command in this course after this
point is written with `sudo` in front of it, exactly as you'd do it on
a real production box. If you're already root, the `sudo` is a harmless
no-op.

**Common error:**
```
your-user is not in the sudoers file. This incident will be reported.
```
**Root cause:** the user isn't in the `wheel` group.
**Fix (as root):**
```bash
usermod -aG wheel your-user
```
Log out and back in for the group membership to take effect.

---

## 5. Hostname — `hostname` vs `hostnamectl`

### What it is

A hostname is the machine's own name. On a DNS server this name will
frequently show up *inside the DNS system itself* (as the target of an
NS record, in SOA records, in TLS certs later, in monitoring dashboards).
Getting it right, and getting it **permanent**, matters far more here
than on an ordinary application server.

### Check the current hostname

```bash
hostname
```
**What it does:** prints the *runtime* hostname the kernel currently
holds in memory.

```bash
hostnamectl
```
**What it does:** `hostnamectl` is the modern systemd tool for viewing
**and setting** hostname-related identity. Run with no arguments, it
shows a full status block:

```
Static hostname: localhost.localdomain
Icon name: computer-vm
Chassis: vm
Machine ID: 8f3a9c1e...
Boot ID: 2b7e44f1...
Virtualization: kvm
Operating System: AlmaLinux 9.4
CPE OS Name: cpe:/o:almalinux:almalinux:9::baseos
Kernel: Linux 5.14.0-427.el9.x86_64
Architecture: x86-64
```

There are actually **three kinds** of hostname in systemd:
- **Static** — the traditional, permanent name stored in `/etc/hostname`
- **Transient** — set by the kernel/DHCP, lost on reboot if not backed by static
- **Pretty** — a free-form display name (e.g. `Arun's DNS Box`), rarely used operationally

For this course, we only care about the **static** hostname.

### Set the hostname permanently

```bash
sudo hostnamectl set-hostname dns01.internal.example.com
```

**What it does:** writes the given FQDN into `/etc/hostname` and tells
`systemd-hostnamed` to update the running kernel hostname immediately —
**no reboot required**.

**Why the FQDN and not just `dns01`:** many services (including BIND
later, and mail/TLS tooling in general) expect `hostname -f` to return a
fully qualified domain name. Setting the *static* hostname to the FQDN
directly is the cleanest way to guarantee this on RHEL-family systems.

### Verify

```bash
hostname
hostname -f
cat /etc/hostname
```

**Expected output:**
```
$ hostname
dns01.internal.example.com

$ hostname -f
dns01.internal.example.com

$ cat /etc/hostname
dns01.internal.example.com
```

**Common error:** `hostname -f` returns just `dns01` or an error like
`hostname: Name or service not known`. **Root cause:** the short
hostname was set instead of the FQDN, and `/etc/hosts` doesn't resolve
it either. **Fix:** re-run `hostnamectl set-hostname` with the full FQDN,
then check `/etc/hosts` (next section).

**Ubuntu/Debian note:** `hostnamectl` behaves identically — it's the
same systemd component. No difference here.

---

## 6. `/etc/hosts` — local, static name resolution

### What it is

`/etc/hosts` is the oldest and simplest name-resolution mechanism on any
Unix-like system: a flat text file mapping IP addresses to names,
checked **before** DNS is even consulted (this ordering is controlled by
`/etc/nsswitch.conf`, which we won't need to touch in this course, but
it's worth knowing it exists).

### Why it matters on a DNS server specifically

If `dns01`'s own hostname doesn't resolve via `/etc/hosts` (or DNS —
which doesn't exist yet, that's what we're building!), some services
that call `gethostname()`-style lookups internally will hang or log
warnings on startup. We fix this **before** BIND is even installed.

### View it

```bash
cat /etc/hosts
```

**Expected default content on a fresh install:**
```
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
```

### Edit it

```bash
sudo vi /etc/hosts
```

Add a line mapping the server's real static IP (we set this IP in
section 7 below — if you're doing this before configuring the static
IP, come back and confirm afterward) to its FQDN and short name:

```
192.168.10.10   dns01.internal.example.com   dns01
```

**Format explained:** `<IP> <FQDN> <short-alias> [more aliases...]` —
the FQDN should come first after the IP; the shorter alias afterward.
Many tools that reverse-map an IP to a name will pick the *first* name
listed.

### Verify

```bash
getent hosts dns01.internal.example.com
ping -c 2 dns01.internal.example.com
```

**What `getent hosts` does:** queries the system's Name Service Switch
(NSS) using whatever sources are configured (files, then DNS, per
`/etc/nsswitch.conf`) — this is the *same lookup path real
applications use*, which makes it a more trustworthy test than manually
reading the file.

**Expected output:**
```
$ getent hosts dns01.internal.example.com
192.168.10.10   dns01.internal.example.com dns01
```

---

## 7. Networking — interfaces, IP, gateway

### Identify your network interface

```bash
ip addr
```
(short form: `ip a`)

**What it does:** lists every network interface and its currently
assigned addresses. On a fresh cloud/VM install you'll typically see
`lo` (loopback, always `127.0.0.1/8`) and one Ethernet-style interface,
commonly named `eth0`, `ens160`, `ens192`, or `enp0s3` depending on the
hypervisor/driver.

**Example output:**
```
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:4a:1c:9e brd ff:ff:ff:ff:ff:ff
    inet 192.168.10.10/24 brd 192.168.10.255 scope global noprefixroute enp0s3
```

Note the interface name — you'll need it (`enp0s3` in this example) for
the next commands.

```bash
ip route
```
**What it does:** shows the kernel routing table, including the default
gateway — the router traffic goes to when the destination isn't on the
local subnet.

**Example output:**
```
default via 192.168.10.1 dev enp0s3 proto static metric 100
192.168.10.0/24 dev enp0s3 proto kernel scope link src 192.168.10.10 metric 100
```

`default via 192.168.10.1` tells you the gateway is `192.168.10.1`.

### Configure networking with NetworkManager / `nmcli`

RHEL-family systems (9.x) manage networking via **NetworkManager** by
default. `nmcli` is its command-line client.

```bash
nmcli connection show
```
**What it does:** lists all NetworkManager "connections" (profiles) —
not the same thing as interfaces; a connection is a saved config that
gets *applied to* an interface.

**Example output:**
```
NAME                UUID                                  TYPE      DEVICE
enp0s3              8b1e4b2a-2222-4c3e-9c11-abcde1234567  ethernet  enp0s3
```

### Set a static IP

```bash
sudo nmcli connection modify enp0s3 \
  ipv4.addresses 192.168.10.10/24 \
  ipv4.gateway 192.168.10.1 \
  ipv4.dns 192.168.10.10 \
  ipv4.method manual
```

**What each flag does:**
- `connection modify enp0s3` — edit the saved connection profile named
  `enp0s3` (use the exact name from `nmcli connection show`)
- `ipv4.addresses` — the static IP and CIDR prefix (`/24` = subnet mask
  `255.255.255.0`)
- `ipv4.gateway` — the default route target
- `ipv4.dns` — the DNS server this machine will use for **its own**
  outbound lookups. We're pointing it at itself (`192.168.10.10`)
  because this machine *will be* the DNS server after Module 05–07.
  Until BIND is installed and running, this will actually cause lookups
  to fail — that's expected and we'll revisit it.
- `ipv4.method manual` — switch off DHCP for IPv4 and use only the
  static values above

Apply the change:

```bash
sudo nmcli connection up enp0s3
```

**What it does:** re-activates the connection profile, applying the new
static configuration immediately (this may briefly drop your SSH
session if you're connected over the interface you just reconfigured —
reconnect using the new static IP).

### Verify

```bash
ip addr show enp0s3
ip route
```

**Expected output:**
```
inet 192.168.10.10/24 brd 192.168.10.255 scope global noprefixroute enp0s3
```
and
```
default via 192.168.10.1 dev enp0s3 proto static metric 100
```

**Common error:**
```
Error: Connection activation failed: IP configuration could not be reserved
```
**Root cause:** usually a duplicate static IP already in use on the
subnet, or a typo in the CIDR prefix.
**Fix:** `ping 192.168.10.10` from another host *before* assigning it,
to confirm it's free; double-check `ipv4.addresses` syntax
(`IP/prefix`, not `IP` and `netmask` separately).

**Ubuntu/Debian note:** modern Ubuntu Server (20.04+) uses **Netplan**
(a YAML front-end, often driving `systemd-networkd` instead of
NetworkManager on servers). The equivalent static config lives in
`/etc/netplan/*.yaml`:
```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: no
      addresses: [192.168.10.10/24]
      routes:
        - to: default
          via: 192.168.10.1
      nameservers:
        addresses: [192.168.10.10]
```
Applied with `sudo netplan apply`. Conceptually identical to the
`nmcli` steps above — same four settings, different tool.

---

## 8. `/etc/resolv.conf` — where *this machine* looks things up

### What it is

`/etc/resolv.conf` tells the local resolver library (`glibc`) which DNS
server(s) to query and which domain(s) to search when you type a short
hostname.

### View it

```bash
cat /etc/resolv.conf
```

On NetworkManager-managed RHEL systems, this file is usually
**auto-generated** — don't hand-edit it directly, because NetworkManager
will overwrite your edits the next time the connection is brought up or
down. It reflects whatever you set with `ipv4.dns` in the `nmcli`
command above.

**Expected output after the `nmcli` change in section 7:**
```
# Generated by NetworkManager
search internal.example.com
nameserver 192.168.10.10
```

### Why this file matters for a DNS-server-in-progress

Right now, `nameserver 192.168.10.10` points at *this very machine*,
but BIND isn't installed yet — so any DNS lookup this machine tries to
do (e.g. `dnf update`, which needs to resolve mirror hostnames) will
**fail** until you either install and start BIND (Module 05) or
temporarily point this file at a working public resolver.

**This is expected and intentional for this course** — but if you need
working internet access *right now* (e.g. to run `dnf install` in
Module 05), temporarily override with a public resolver:

```bash
sudo nmcli connection modify enp0s3 ipv4.dns "8.8.8.8 1.1.1.1"
sudo nmcli connection up enp0s3
```

Then switch it back to `192.168.10.10` once BIND is running in Module 05.

**Interview-relevant distinction to internalize now:** `ipv4.dns` /
`/etc/resolv.conf` on the DNS server controls what *that host* queries.
It has **nothing to do with** what BIND *answers* to other clients —
that's entirely governed by `named.conf` and zone files, which we build
starting Module 06. Conflating these two is a very common junior-level
mistake.

---

## 9. Firewall — `firewalld`

We are not opening any ports yet (that's Module 19, once BIND actually
exists) — but you must know the service exists and how to check its
state now, because a "DNS isn't working" symptom later is very often
"firewall is blocking port 53," and you'll want `firewall-cmd` reflexes
already built.

```bash
systemctl status firewalld
```

**What it does:** shows whether the `firewalld` service is active,
enabled at boot, and its recent log lines.

**Expected output (fresh install, firewalld active by default on RHEL-family):**
```
● firewalld.service - firewalld - dynamic firewall daemon
     Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; preset: enabled)
     Active: active (running) since ...
```

```bash
sudo firewall-cmd --state
```
**What it does:** a quick one-word check — `running` or `not running` —
useful in scripts (we'll use this in Module 25/26 automation).

**Ubuntu/Debian note:** Ubuntu typically ships **ufw** (Uncomplicated
Firewall) instead of firewalld, often disabled by default on server
installs. `sudo ufw status` is the equivalent check. We'll cover the
`ufw` equivalents for DNS ports in Module 19 alongside `firewall-cmd`.

---

## 10. SELinux

```bash
getenforce
```
**What it does:** prints the current SELinux mode: `Enforcing`,
`Permissive`, or `Disabled`.

**Expected output on a default AlmaLinux/Rocky install:**
```
Enforcing
```

```bash
sestatus
```
**What it does:** a more detailed status block — policy version, mode,
mount point, etc.

**Do not disable SELinux for this course.** A large part of Module 18
is specifically about correctly running BIND *with* SELinux enforcing —
that's a real production skill and disabling SELinux to "make it work"
is exactly the anti-pattern this course avoids. If you find yourself
tempted to run `setenforce 0` as a fix at any later point, treat that as
a sign you've missed a step, not a solution.

**Ubuntu/Debian note:** Ubuntu Server typically ships **AppArmor**
instead of SELinux. `sudo aa-status` is the rough equivalent. This
course's SELinux module (18) is RHEL-specific; if you're following along
on Ubuntu, note that AppArmor profiles for `bind9` exist but the
troubleshooting commands differ — this course won't cover AppArmor in
depth.

---

## 11. Time synchronization

### Why this matters for DNS specifically

DNS relies on **serial numbers and TTLs**, not wall-clock time directly,
so DNS itself is fairly time-tolerant. However: DNSSEC (touched on in
Module 17) is **not** time-tolerant — signature validity windows depend
on accurate system clocks. Get this right now so it's a non-issue later.

```bash
timedatectl
```
**What it does:** shows local time, timezone, and whether NTP
synchronization is active.

**Expected output:**
```
               Local time: Sat 2026-08-22 14:32:10 IST
           Universal time: Sat 2026-08-22 09:02:10 UTC
                 RTC time: Sat 2026-08-22 09:02:10
                Time zone: Asia/Kolkata (IST, +0530)
System clock synchronized: yes
              NTP service: active
          RTC in local TZ: no
```

If `NTP service` shows `inactive`:

```bash
sudo timedatectl set-ntp true
```
**What it does:** enables `chronyd` (RHEL-family's default NTP client)
to keep the clock synced automatically.

Set the timezone if needed:
```bash
sudo timedatectl set-timezone Asia/Kolkata
```

---

## 12. Package management basics you'll rely on for the rest of the course

RHEL-family uses `dnf` (the successor to `yum`, which still works as an
alias).

```bash
sudo dnf update -y
```
**What it does:** refreshes package metadata and upgrades all installed
packages to their latest available version. `-y` auto-confirms prompts.
Run this once now on a fresh box, before Module 05's `dnf install bind
bind-utils`.

```bash
sudo dnf install -y vim
```
**What it does:** installs a package — shown here with `vim` as an
example since you'll be editing config files by hand throughout this
course (`vi`/`vim` ships by default on most minimal installs, but
confirm it's present).

**Ubuntu/Debian note:** the equivalent commands are `sudo apt update`
and `sudo apt install -y <package>`.

---

## 13. `systemctl` and `journalctl` — your two most-used commands from here on

Every service in this course (`NetworkManager`, `firewalld`, and
starting Module 05, `named`) is managed via `systemd`.

```bash
systemctl status NetworkManager
```
**What it does:** shows whether a service is `active (running)`,
`enabled` (starts at boot) or `disabled`, plus its most recent log
lines inline — often enough to spot a problem without a separate log
command.

**The states you need to recognize:**
| State | Meaning |
|---|---|
| `active (running)` | service is up and healthy |
| `inactive (dead)` | service is stopped |
| `failed` | service tried to start and crashed/errored |
| `enabled` | will auto-start on boot |
| `disabled` | will **not** auto-start on boot |

```bash
journalctl -u NetworkManager
```
**What it does:** shows the full systemd journal (log) for a specific
unit (`-u`). This is the primary log tool you will use in Module 21
(Troubleshooting) to diagnose why `named` failed to start.

Useful variants to know now:
```bash
journalctl -u NetworkManager -f     # follow live, like `tail -f`
journalctl -u NetworkManager -n 50  # last 50 lines only
journalctl -u NetworkManager --since "10 min ago"
```

---

## What I Learned

- How to permanently set an FQDN hostname with `hostnamectl` and verify
  it with `hostname -f`
- Why `/etc/hosts` matters even on a machine that will itself become a
  DNS server
- How to convert a DHCP interface to a static IP with `nmcli`
  (and the Netplan equivalent on Ubuntu)
- The distinction between what a server's `/etc/resolv.conf` resolves
  *for itself* versus what BIND will later *serve to others*
- How to check (not yet configure) `firewalld` and SELinux state, and
  why we never disable SELinux as a "fix"
- Why accurate time matters later for DNSSEC
- The `systemctl` states and `journalctl` flags you'll reuse in every
  future module

## Commands to Remember

```bash
hostnamectl set-hostname dns01.internal.example.com
hostname -f
cat /etc/hosts
ip addr
ip route
nmcli connection show
nmcli connection modify <iface> ipv4.addresses <ip>/<prefix> ipv4.gateway <gw> ipv4.dns <dns> ipv4.method manual
nmcli connection up <iface>
cat /etc/resolv.conf
systemctl status firewalld
firewall-cmd --state
getenforce
sestatus
timedatectl
dnf update -y
systemctl status <service>
journalctl -u <service> -f
```

## Practical Lab

Build **DNS01** from a fresh AlmaLinux/Rocky minimal install:

1. Set the hostname to `dns01.internal.example.com` and confirm
   `hostname -f` returns the FQDN.
2. Add the correct line to `/etc/hosts` and confirm with
   `getent hosts dns01.internal.example.com`.
3. Convert the interface to a static IP `192.168.10.10/24`, gateway
   `192.168.10.1`, DNS `192.168.10.10`, using `nmcli`.
4. Confirm the new IP with `ip addr` and the default route with
   `ip route`.
5. Confirm `firewalld` is active and note it down — don't open any
   ports yet.
6. Confirm SELinux is `Enforcing`.
7. Confirm NTP sync is active with `timedatectl`.
8. Run `sudo dnf update -y` successfully (temporarily point
   `ipv4.dns` at `8.8.8.8` first if this machine can't resolve yet).

## Troubleshooting Exercise

Your hostname was set with `hostnamectl set-hostname dns01` (short name
only, not FQDN). Diagnose and fix:

1. **Symptom:** `hostname -f` returns `dns01` with no domain suffix, or
   an error.
2. **Diagnosis:** `cat /etc/hostname` — confirm it only contains the
   short name.
3. **Commands:** `hostnamectl`, `cat /etc/hostname`, `cat /etc/hosts`
4. **Root cause:** the static hostname was set without the domain
   portion, so nothing on the system can qualify it into an FQDN.
5. **Fix:** `sudo hostnamectl set-hostname dns01.internal.example.com`
6. **Verification:** `hostname -f` returns the full FQDN;
   `getent hosts dns01.internal.example.com` resolves correctly.

## Interview Questions

- **What's the difference between the "static," "transient," and
  "pretty" hostname in systemd?**
  Short answer: static is the permanent name in `/etc/hostname`;
  transient is set at runtime (e.g. by DHCP) and doesn't persist unless
  no static name is set; pretty is a free-form display label. Detailed:
  `hostnamectl` manages all three but for infrastructure purposes only
  the static hostname matters — it's what persists across reboots and
  what most services read. Real-world example: cloud-init on an AWS EC2
  instance often sets a transient hostname from DHCP; without a static
  hostname set explicitly, that name can change on stop/start, breaking
  anything (like a DNS NS record) that assumed a stable name.

- **Why does `/etc/resolv.conf` on a DNS server matter, given that the
  server itself serves DNS to others?**
  Short answer: it controls the resolver behavior of *that specific
  machine's own outbound lookups*, completely separate from what BIND
  answers to clients. Detailed: BIND's answers are governed by
  `named.conf` and zone files; `/etc/resolv.conf` (or its NetworkManager
  source) governs what the OS's own `glibc` resolver does for things
  like `dnf`, `curl`, or `ping` run *on that host*. Real-world example:
  pointing a DNS server's own `resolv.conf` at itself before BIND is
  installed and running will break that server's own outbound internet
  access — a very common "day one" gotcha.

- **Why avoid disabling SELinux instead of troubleshooting it?**
  Short answer: disabling SELinux removes a real security control
  permanently rather than fixing the actual misconfiguration. Detailed:
  SELinux denials are diagnosable (`ausearch`, `sealert`, covered in
  Module 18) and fixable with targeted booleans or contexts
  (`semanage`, `restorecon`) that keep protection intact. Real-world
  example: production RHEL environments are frequently subject to
  compliance requirements (e.g. STIG, CIS benchmarks) that mandate
  SELinux stay enforcing — "just disable it" is not an option available
  to a production engineer.

## Expected Result

At the end of this module you have one Linux server, **DNS01**, that:
- Answers to `hostname -f` → `dns01.internal.example.com`
- Has a static IP `192.168.10.10/24` with gateway `192.168.10.1`
- Resolves its own name locally via `/etc/hosts`
- Has `firewalld` running (unconfigured for DNS — that's Module 19)
- Has SELinux `Enforcing`
- Has NTP time sync active
- Is fully patched (`dnf update -y` completed)
- You're comfortable running `systemctl status` / `journalctl -u` against
  any service

This server is now ready for **MODULE 02 — Linux Networking
Fundamentals**, where we go one level deeper into the TCP/UDP concepts
DNS itself depends on, before installing BIND in Module 05.

---

Say **NEXT** to continue to Module 02.
