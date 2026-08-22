# MODULE 02 — Linux Networking Fundamentals

> Course: BIND DNS from Zero | Distro: AlmaLinux / Rocky / RHEL (Ubuntu differences noted)
> Builds on: **DNS01** — `192.168.10.10` (Module 01)

DNS is fundamentally a network protocol running on top of the same
IP/TCP/UDP stack as everything else. Before you configure BIND, you
need to be fluent in the networking primitives DNS depends on:
addressing, ports, sockets, and — most importantly — *why DNS mostly
uses UDP but sometimes needs TCP*. This module builds that fluency with
hands-on commands, not just definitions.

---

## 1. What this module covers

- IPv4 addressing, subnet masks, CIDR
- IPv6 basics (enough to recognize AAAA records later)
- Network address vs broadcast address vs gateway
- Routing, at a conceptual level
- TCP vs UDP — and specifically why DNS uses UDP port 53 (and TCP port 53)
- Ports and sockets
- Loopback / localhost
- Hands-on tools: `ip`, `ping`, `traceroute`/`tracepath`, `ss`, `curl`, `nc`, `tcpdump`

## 2. Why this matters before touching BIND

Almost every DNS troubleshooting session (Module 21) reduces to one of
these questions:
- Is the packet leaving the client at all? (`ip route`, `ping`)
- Is the packet reaching the DNS server? (`traceroute`, `tcpdump`)
- Is anything actually listening on port 53? (`ss`)
- Is the response coming back over the protocol I expect (UDP small
  answer vs TCP large answer)?

If you can't answer these independently of DNS itself, every DNS bug
looks like a DNS bug — even when the real cause is routing or a
firewall. This module builds that independent diagnostic layer.

---

## 3. IPv4 addressing, subnet masks, CIDR

### What it is

An IPv4 address is a 32-bit number, written as four decimal octets
(`192.168.10.10`), each 0–255. A **subnet mask** (or its CIDR shorthand,
`/24`) tells you how many of those 32 bits identify the *network*
versus how many identify the *host* on that network.

### Why it matters here

Our lab subnet is `192.168.10.0/24`:
- `/24` = 255.255.255.0 = the first 24 bits are the network portion
- That leaves 8 bits for host addresses: `.0` through `.255`
- `.0` is the **network address** (identifies the subnet itself, not a
  host)
- `.255` is the **broadcast address** (sent to reach every host on the
  subnet at once)
- Usable host addresses: `.1` through `.254`

This is exactly why DNS01 (`.10`), DNS02 (`.11`), WEB01 (`.20`), DB01
(`.30`), and CLIENT01 (`.50`) all fit inside one `/24` — with room to
grow.

### See it in practice

```bash
ip addr show
```
Confirms DNS01's address again: `192.168.10.10/24`. The `/24` here is
doing exactly the job described above.

**CIDR quick-reference table** (you'll see these in real infrastructure
constantly):

| CIDR | Subnet mask | Usable hosts |
|---|---|---|
| /24 | 255.255.255.0 | 254 |
| /25 | 255.255.255.128 | 126 |
| /26 | 255.255.255.192 | 62 |
| /28 | 255.255.255.240 | 14 |
| /30 | 255.255.255.252 | 2 (common for point-to-point links) |

**Common mistake:** assigning a static IP with the wrong prefix (e.g.
`/16` instead of `/24`) — the host still gets an address, but it
computes the wrong broadcast/network boundaries, and routing or ARP
behavior can silently break. Always double check the prefix matches
what the rest of your subnet is actually using.

---

## 4. IPv6 basics (just enough for this course)

You won't deploy IPv6 in the lab, but you need to recognize it because
Module 10 covers **AAAA records** (IPv6's equivalent of an A record).

- IPv6 addresses are 128 bits, written in 8 groups of hex, e.g.
  `2001:0db8:0000:0000:0000:0000:0000:0001`, commonly compressed to
  `2001:db8::1` (the `::` collapses one run of zero groups, only once
  per address).
- `::1` is IPv6's loopback address — the equivalent of `127.0.0.1`.
- `fe80::/10` is the link-local prefix — every interface has one
  automatically; it's not globally routable.

```bash
ip -6 addr show
```
**What it does:** shows IPv6 addresses assigned to each interface —
you'll typically see at least a `fe80::` link-local address even if you
never configured IPv6 manually.

---

## 5. Network address, gateway, and routing

### What it is

When your machine needs to send a packet, it consults its **routing
table** to decide: is the destination on my local subnet (deliver
directly via ARP), or is it elsewhere (send to the default gateway,
which forwards it onward)?

```bash
ip route
```
**Expected output on DNS01:**
```
default via 192.168.10.1 dev enp0s3 proto static metric 100
192.168.10.0/24 dev enp0s3 proto kernel scope link src 192.168.10.10 metric 100
```

**Reading this line by line:**
- `192.168.10.0/24 dev enp0s3 ... scope link` — "anything in this
  subnet, reach directly through this interface, no gateway needed."
  This route is added automatically the moment you assign an IP in
  that subnet.
- `default via 192.168.10.1` — "anything *not* covered by a more
  specific route, send to `192.168.10.1`" (the gateway/router).

**Why this matters for DNS:** when DNS01 later needs to forward a query
to `8.8.8.8` (Module 12), this routing table — not BIND — is what
decides the packet leaves via the gateway.

---

## 6. TCP vs UDP — and why DNS cares deeply about this distinction

This is one of the single most commonly asked DNS interview topics, so
we go deep here.

### UDP (User Datagram Protocol)

- **Connectionless** — no handshake, just fire a packet and hope it
  arrives
- **No retransmission** built into the protocol itself — if it's lost,
  it's lost (the application layer, i.e. the DNS resolver, has to
  retry)
- **Low overhead, fast** — a single round trip is enough for most
  queries
- **Size-limited** in the traditional (pre-EDNS0) sense — historically
  512 bytes per UDP DNS message

### TCP (Transmission Control Protocol)

- **Connection-oriented** — three-way handshake (SYN, SYN-ACK, ACK)
  before any data flows
- **Reliable** — lost segments are automatically retransmitted, order
  guaranteed
- **Higher overhead** — more round trips, more state to track
- **No inherent size limit** for a single message the way classic UDP
  DNS had

### Why DNS defaults to UDP

A typical DNS query/response is small (a hostname in, an IP address
out) and needs to be **fast** — DNS resolution sits in the critical
path of nearly every network operation a client does. The overhead of
a full TCP handshake for a single small lookup would meaningfully slow
down the internet in aggregate. UDP's fire-and-forget model, with the
resolver handling retries itself, is a better fit for this pattern.

### When DNS falls back to TCP

1. **Large responses** — when the answer exceeds what fits in a single
   UDP datagram (particularly relevant with DNSSEC-signed responses,
   which are much larger; also relevant with `TXT` records containing
   long strings).
2. **Zone transfers (AXFR/IXFR)** — Module 15. Transferring an entire
   zone from a primary to a secondary server is not a "one small
   answer" operation — it needs TCP's reliability and lack of size
   limit.
3. **Truncation** — if a UDP response would exceed the size limit, the
   server sets the **TC (truncated) flag** in its reply; the client is
   expected to notice this and re-issue the same query over TCP.
4. **EDNS0** partially raises the effective UDP size limit (advertised
   via the OPT pseudo-record) without needing TCP, but very large
   answers — and zone transfers specifically — still require TCP.

**This is why, starting Module 19, you'll open *both* UDP 53 *and* TCP
53 in the firewall** — opening only UDP 53 is one of the most common
real-world DNS misconfigurations, and it manifests as "small lookups
work fine, but zone transfers or DNSSEC-heavy domains fail" — a
confusing, hard-to-diagnose symptom if you don't already know this
UDP/TCP relationship going in.

---

## 7. Ports and sockets

### What it is

A **port** is a 16-bit number (0–65535) that lets a single IP address
host multiple independent network services simultaneously. A **socket**
is the combination of `(protocol, local IP, local port, remote IP,
remote port)` that uniquely identifies one active network conversation.

**DNS's well-known port: 53** — both UDP/53 and TCP/53, as covered
above. This is a IANA-registered "well-known port" (0–1023 range),
which is also why `named` needs elevated privileges (or specific
capabilities) to bind to it — a detail that becomes relevant in Module
17/18's security discussion.

### `localhost` and loopback

`127.0.0.1` (IPv4) / `::1` (IPv6) always refers to "this machine,"
routed entirely within the kernel's own network stack — packets never
touch a physical interface. `localhost` is the hostname that resolves
to it (via `/etc/hosts`, which you saw in Module 01).

```bash
ping -c 2 localhost
```
**What it does:** confirms the loopback interface is functioning —
useful as a first sanity check before blaming DNS or the network for a
problem that might just be "the service isn't running at all."

---

## 8. Hands-on: `ping`

```bash
ping -c 4 192.168.10.1
```
**What it does:** sends 4 ICMP Echo Request packets to the gateway and
waits for Echo Replies. `-c 4` limits it to 4 packets (without `-c`,
`ping` runs forever until you `Ctrl+C`).

**Expected output:**
```
PING 192.168.10.1 (192.168.10.1) 56(84) bytes of data.
64 bytes from 192.168.10.1: icmp_seq=1 ttl=64 time=0.412 ms
64 bytes from 192.168.10.1: icmp_seq=2 ttl=64 time=0.389 ms
64 bytes from 192.168.10.1: icmp_seq=3 ttl=64 time=0.401 ms
64 bytes from 192.168.10.1: icmp_seq=4 ttl=64 time=0.377 ms

--- 192.168.10.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3003ms
rtt min/avg/max/mdev = 0.377/0.412/0.389/0.014 ms
```

**Important limitation to internalize now:** `ping` tests basic IP
reachability using **ICMP**, a completely different protocol from
UDP/TCP. A host can block ICMP (no ping response) while still happily
serving DNS over UDP/TCP 53 — so "I can't ping the DNS server" is
*not* proof that DNS itself is broken, and "I can ping it" is *not*
proof that port 53 is actually open. You'll use `ss` and `dig`/`nc` for
that (below, and Module 04/21).

---

## 9. Hands-on: `traceroute` and `tracepath`

```bash
traceroute 8.8.8.8
```
**What it does:** discovers the path (each router hop) a packet takes
to reach a destination, by sending packets with increasing TTL (Time To
Live) values and recording which router replies "TTL exceeded" at each
step.

**Example output:**
```
traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets
 1  192.168.10.1 (192.168.10.1)  0.512 ms  0.480 ms  0.455 ms
 2  10.0.0.1 (10.0.0.1)  2.114 ms  2.098 ms  2.075 ms
 3  * * *
 4  142.250.xxx.xxx  12.331 ms  12.290 ms  12.245 ms
 8  8.8.8.8 (8.8.8.8)  14.882 ms  14.801 ms  14.750 ms
```

`* * *` means that particular hop didn't reply (often intentional
firewall/rate-limiting on transit routers) — this is normal and not
itself a failure, as long as later hops still respond.

**If `traceroute` isn't installed:**
```bash
sudo dnf install -y traceroute
```

```bash
tracepath 8.8.8.8
```
**What it does:** a lighter alternative to `traceroute` that doesn't
require root privileges (uses UDP with increasing TTL, similar
principle) and is installed by default alongside `iproute` on most
RHEL-family minimal installs.

**Why this matters for DNS troubleshooting:** when a client can't reach
an external forwarder (`8.8.8.8`, Module 12) or a remote authoritative
server, `traceroute`/`tracepath` tells you *where* the path breaks —
locally, at your gateway, or somewhere upstream — before you waste time
looking at BIND config that might be perfectly fine.

---

## 10. Hands-on: `ss` — socket statistics

This is one of the single most important commands for the rest of this
course. You will use it constantly starting Module 05 to confirm
`named` is actually listening.

```bash
sudo ss -lntup
```

**Flag-by-flag breakdown:**
- `-l` — show only **listening** sockets (services waiting for
  connections), not established connections
- `-n` — numeric output (show port numbers and IPs, don't resolve
  service names or hostnames — faster, and avoids DNS-dependent output
  which would be circular on a DNS server anyway)
- `-t` — show TCP sockets
- `-u` — show UDP sockets
- `-p` — show the **process** owning each socket (requires `sudo` to
  see process names/PIDs for sockets you don't own)

**Example output once BIND is installed (Module 05 preview):**
```
Netid State  Recv-Q Send-Q Local Address:Port  Peer Address:Port Process
udp   UNCONN 0      0      192.168.10.10:53    0.0.0.0:*         users:(("named",pid=1234,fd=20))
tcp   LISTEN 0      10     192.168.10.10:53    0.0.0.0:*         users:(("named",pid=1234,fd=21))
```

Two lines, same port 53, one `udp`/`UNCONN` (UDP doesn't have a
"listening" state the way TCP does — `UNCONN` is the closest UDP
equivalent) and one `tcp`/`LISTEN` — exactly matching the UDP+TCP
duality explained in section 6.

**Right now, before BIND exists,** run this anyway as a baseline:
```bash
sudo ss -lntup
```
You should **not** see anything on port 53 yet — confirming a clean
slate before Module 05 installs BIND. If something else is already
listening on 53 (rare, but possible if `systemd-resolved` is active,
common on some Ubuntu installs), you'll need to disable it before BIND
can bind to the port — this exact conflict is covered as a named
troubleshooting scenario in Module 21.

---

## 11. Hands-on: `curl` and `nc` for manual connectivity testing

```bash
curl -v telnet://192.168.10.1:53
```
This is a slightly unusual use of `curl`, but it's a quick way to test
whether a TCP port is open without installing extra tools — `curl`
attempts a raw TCP connection and reports success/failure of the
connect step, then you can `Ctrl+C` out.

More directly, **`nc` (netcat)** is the standard tool for this:

```bash
nc -zv 192.168.10.10 53
```
**Flag-by-flag:**
- `-z` — "zero-I/O mode": just test whether the connection can be made,
  don't actually send/receive data
- `-v` — verbose, print the result

**Expected output once something is listening:**
```
Connection to 192.168.10.10 53 port [tcp/domain] succeeded!
```
(`domain` is `/etc/services`'s registered name for port 53 — you'll see
this in a lot of tool output throughout the course.)

**Expected output right now (nothing listening yet):**
```
nc: connect to 192.168.10.10 port 53 (tcp) failed: Connection refused
```
This is the correct, expected result at this stage of the course —
confirm it and move on.

**Important limitation:** `nc -z` by default only tests TCP. For UDP:
```bash
nc -zvu 192.168.10.10 53
```
UDP connectivity tests are inherently less reliable (UDP has no
handshake to confirm success), so `nc -zvu` often reports "succeeded"
even when nothing is actually listening — treat a UDP `nc` result as a
weak signal, and always cross-check with `ss -lntup` (which tells you
definitively whether something is bound to the port) or, once BIND is
running, an actual `dig` query (Module 04).

---

## 12. Hands-on: `tcpdump` — see the actual packets

```bash
sudo tcpdump -i enp0s3 port 53 -n
```

**Flag-by-flag:**
- `-i enp0s3` — capture on this interface specifically (use the
  interface name from `ip addr`)
- `port 53` — a Berkeley Packet Filter (BPF) expression, capture only
  traffic to/from port 53 (both UDP and TCP, both directions)
- `-n` — don't resolve IP addresses to hostnames (again, avoids
  DNS-dependent output on a DNS server)

Once BIND is running and answering queries (Module 05 onward), this is
your ground-truth tool: it shows you *exactly* what's on the wire,
independent of any assumption `dig` or `named`'s logs might be making.
You'll come back to this constantly in Module 21's troubleshooting
methodology — "is the query even reaching the server" is always
answerable with `tcpdump`, even when every other layer of the stack is
lying to you or misconfigured.

**Common flags you'll add later:**
```bash
sudo tcpdump -i enp0s3 port 53 -n -w /tmp/dns-capture.pcap   # write to a file for later analysis
sudo tcpdump -i enp0s3 port 53 -n -A                          # show packet contents in ASCII
```

---

## 13. Practical exercise: put it all together

Right now, on DNS01, with BIND not yet installed, run this sequence and
interpret each result:

```bash
ip addr
ip route
ping -c 3 192.168.10.1
traceroute 8.8.8.8
sudo ss -lntup | grep ':53' || echo "Nothing listening on port 53 yet — expected"
nc -zv 192.168.10.10 53
```

You should see: a static IP and correct route (from Module 01), a
successful ping to the gateway, a traceroute that eventually reaches
`8.8.8.8`, **nothing** on port 53 in `ss`, and a `Connection refused`
from `nc`. If any of these differ from expectations, resolve it now —
Module 05 assumes this exact clean baseline.

---

## What I Learned

- How CIDR notation (`/24`) determines network address, broadcast
  address, and usable host range
- The practical difference between UDP (fast, connectionless, DNS's
  default) and TCP (reliable, used for zone transfers, large/truncated
  DNS responses) — and *why* this split exists
- Why `ping` (ICMP) reachability is not the same thing as a port
  actually being open
- How to read a routing table with `ip route`
- How to confirm what's listening on a port with `ss -lntup` — the
  single most important verification command for the rest of this
  course
- How to manually test TCP/UDP port connectivity with `nc`
- How to capture and inspect raw traffic with `tcpdump`

## Commands to Remember

```bash
ip addr
ip -6 addr show
ip route
ping -c 4 <ip>
traceroute <ip>
tracepath <ip>
sudo ss -lntup
nc -zv <ip> <port>
nc -zvu <ip> <port>
sudo tcpdump -i <iface> port 53 -n
```

## Practical Lab

On DNS01 (before BIND is installed):

1. Confirm your subnet math: for `192.168.10.0/24`, write down the
   network address, broadcast address, and usable host range without
   using a calculator — then verify with `ipcalc 192.168.10.0/24` if
   available (`sudo dnf install -y ipcalc`).
2. Run `ip route` and identify which line is the "local subnet, no
   gateway needed" route and which is the default gateway route.
3. `ping -c 3` the gateway, then `traceroute` to `8.8.8.8`, and note
   how many hops it takes.
4. Confirm nothing is listening on port 53 with `ss -lntup`.
5. Start a `tcpdump -i <iface> port 53 -n` capture in one terminal,
   then in a second terminal run `nc -zv 192.168.10.10 53` — observe
   whether tcpdump captures anything (it should, even for a refused
   connection — the SYN packet still goes out).

## Troubleshooting Exercise

You suspect a colleague accidentally left `systemd-resolved` bound to
port 53 on DNS01, which will conflict with BIND once installed in
Module 05.

1. **Symptom:** (anticipated) BIND fails to start later with an
   "address already in use" error.
2. **Diagnosis (do this now, proactively):**
   ```bash
   sudo ss -lntup | grep ':53'
   systemctl status systemd-resolved
   ```
3. **Commands:** `ss -lntup`, `systemctl status systemd-resolved`,
   `systemctl is-enabled systemd-resolved`
4. **Root cause (if found):** `systemd-resolved` is active and has
   bound to `127.0.0.53:53` (its typical stub listener address).
5. **Fix:** on RHEL-family minimal installs this service usually isn't
   present/active by default, so you likely find nothing — document
   that finding. If you *are* on a system where it's active (more
   common on some Ubuntu configurations), disable it:
   ```bash
   sudo systemctl disable --now systemd-resolved
   ```
6. **Verification:** re-run `ss -lntup | grep ':53'` and confirm the
   port is free before proceeding to Module 05.

## Interview Questions

- **Why does DNS use UDP instead of TCP?**
  Short answer: UDP's low overhead (no handshake) suits DNS's typical
  pattern of small, latency-sensitive queries. Detailed: a full TCP
  three-way handshake adds round trips that would meaningfully slow
  down the enormous volume of small lookups DNS handles globally; UDP
  trades guaranteed delivery for speed, and the resolver itself handles
  retries. Real-world example: a web page load can trigger dozens of
  DNS lookups (page domain, CDN, ad networks, fonts, analytics) — even
  a few extra milliseconds per lookup from TCP overhead compounds into
  noticeably slower page loads.

- **When does DNS use TCP?**
  Short answer: for zone transfers, and for responses too large for a
  single UDP datagram (truncated responses, often DNSSEC-heavy
  answers). Detailed: the server sets the TC flag on a UDP response
  that would overflow the size limit, telling the client to re-ask over
  TCP; AXFR/IXFR (Module 15) always use TCP because reliability and
  ordering matter for transferring an entire zone. Real-world example:
  a firewall configured to allow only UDP/53 outbound will cause zone
  transfers between a primary and secondary DNS server to silently
  fail while ordinary lookups keep working fine — a classic
  hard-to-diagnose production incident.

- **What's the practical difference between `ping` succeeding and a
  service actually being reachable?**
  Short answer: `ping` only tests ICMP reachability, not whether any
  particular TCP/UDP port is open or a service is listening. Detailed:
  many production firewalls intentionally block ICMP while still
  allowing specific application traffic (or vice versa), so `ping`
  results and real service reachability can diverge in either
  direction. Real-world example: a DNS server behind a strict firewall
  might not respond to `ping` at all, yet answer DNS queries perfectly
  fine on port 53 — relying on `ping` alone to "check if the server is
  up" would produce a false negative.

## Expected Result

You can now independently verify, without any DNS-specific tooling:
- Your subnet's network/broadcast/host-range math
- Whether your routing table sends traffic where you expect
- Whether a given host is reachable at the IP layer (`ping`)
- Whether a specific TCP or UDP port is actually open (`ss`, `nc`)
- What's actually on the wire at the packet level (`tcpdump`)

This independent networking fluency is what lets you separate "DNS
problem" from "network problem" in every later troubleshooting module.

DNS01 is confirmed to have a clean, port-53-free baseline, ready for
**MODULE 03 — DNS Fundamentals**, where we finally start talking about
DNS itself: what it is, why it exists, and how resolution actually
works end to end — still before installing any software.

---

Say **NEXT** to continue to Module 03.
