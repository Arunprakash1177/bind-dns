# BIND DNS — A Complete Hands-On DevOps/SysEngineer Course

A from-zero, production-oriented course on deploying, operating, and securing
BIND DNS on RHEL-family Linux (AlmaLinux / Rocky / RHEL), built as a
portfolio-grade GitHub repository.

This is **not** a theory article. Every module follows the same pattern:
what it is → why it's needed → how it works → architecture → exact commands
→ expected output → verification → common errors → troubleshooting →
production usage → interview questions.

## Who this is for

Anyone rebuilding their infrastructure fundamentals from a blank Linux
server up to a two-server (primary/secondary) authoritative + recursive
BIND deployment, with monitoring, backup, automation, and security baked in.

## Lab domain and topology

Used consistently across every module:

```
Domain: internal.example.com
Subnet: 192.168.10.0/24

DNS01  (Primary)     192.168.10.10
DNS02  (Secondary)   192.168.10.11
WEB01                192.168.10.20
DB01                 192.168.10.30
GIT01                192.168.10.40
CLIENT01             192.168.10.50
```

## Course progression

| # | Module | Status |
|---|--------|--------|
| 01 | [Linux Server Preparation](docs/01-linux-preparation.md) | ✅ |
| 02 | [Linux Networking Fundamentals](docs/02-networking.md) | ✅ |
| 03 | [DNS Fundamentals](docs/03-dns-fundamentals.md) | ✅ |
| 04 | [DNS Tools](docs/04-dns-tools.md) | ✅ |
| 05 | [Install BIND](docs/05-bind-installation.md) | ✅ |
| 06 | [Basic BIND Configuration](docs/06-bind-configuration.md) | ✅ |
| 07 | [Primary Authoritative DNS](docs/07-primary-dns.md) | ✅ |
| 08 | [DNS Zone Files Deep Dive](docs/08-zone-files.md) | ✅ |
| 09 | Reverse DNS | ⏳ |
| 10 | DNS Record Types | ⏳ |
| 11 | DNS Resolution and Recursion | ⏳ |
| 12 | DNS Forwarding | ⏳ |
| 13 | Caching DNS | ⏳ |
| 14 | Secondary / Slave DNS | ⏳ |
| 15 | AXFR and IXFR | ⏳ |
| 16 | DNS NOTIFY | ⏳ |
| 17 | DNS Security | ⏳ |
| 18 | SELinux + BIND | ⏳ |
| 19 | Firewall + BIND | ⏳ |
| 20 | DNS Logging | ⏳ |
| 21 | DNS Troubleshooting | ⏳ |
| 22 | DNS Client Configuration | ⏳ |
| 23 | High Availability DNS | ⏳ |
| 24 | DNS in DevOps | ⏳ |
| 25 | DNS Automation | ⏳ |
| 26 | Monitoring | ⏳ |
| 27 | Backup and Restore | ⏳ |
| 28 | Production DNS Architecture | ⏳ |
| 29 | Complete Real-World Lab | ⏳ |
| 30 | Interview Preparation | ⏳ |

## Repository layout

```
bind-dns/
├── README.md
├── docs/            # One markdown file per module
├── configs/          # Reusable named.conf / zone file snippets
│   ├── primary/
│   ├── secondary/
│   ├── forward-zone/
│   └── reverse-zone/
├── scripts/          # dns-health-check.sh, dns-backup.sh, dns-restore.sh
├── ansible/           # Automation for Module 25
│   ├── inventory/
│   ├── playbooks/
│   └── roles/
├── labs/              # Standalone lab exercises
│   ├── lab-01/
│   ├── lab-02/
│   └── final-lab/
└── diagrams/          # Architecture diagrams
```

## How to use this course

Work through `docs/` in numeric order. Each module ends with a **Practical
Lab** and a **Troubleshooting Exercise** — do them on a real (or virtual)
AlmaLinux/Rocky machine before moving to the next module. Don't skip ahead;
later modules assume the previous ones' lab state exists.
