# Linux System Administrator Interview Workbook (10 Days)

A practical, scenario-driven Linux System Administrator interview preparation workbook designed for candidates who already understand Linux fundamentals and have completed RHCSA-level training.

This repository focuses on **real-world administration tasks**, **production troubleshooting**, and **interview thinking patterns** rather than memorizing commands or question-answer dumps.

---

## Objective

Most Linux interview preparation material follows a simple pattern:

* Learn commands
* Memorize answers
* Repeat interview questions

Real system administration does not work that way.

In production environments, administrators are expected to:

* Diagnose problems
* Investigate logs
* Analyze symptoms
* Apply fixes with minimal downtime
* Communicate findings
* Follow change-management procedures

This workbook follows the philosophy:

```text
Learn → Apply → Troubleshoot → Explain
```

The goal is to help candidates think like a Linux System Administrator rather than a certification candidate.

---

# Target Audience

This workbook is designed for:

* RHCSA-certified candidates
* Linux Administrators (0–5 Years)
* Infrastructure Engineers
* Support Engineers transitioning to Linux Administration
* DevOps beginners strengthening Linux fundamentals
* Candidates preparing for System Administrator interviews

---

# Learning Methodology

Every topic follows the same structure:

```text
Quick Learning
      ↓
Implementation Lab
      ↓
Production Scenario
      ↓
Troubleshooting Exercise
      ↓
Interview Discussion
```

Instead of memorizing commands, you will learn:

* Why administrators perform a task
* How systems behave under failure
* How to investigate issues
* How to explain your actions during interviews

---

# Workbook Structure

## Day 0 – Linux Foundation Revision

* Linux architecture overview
* RHCSA rapid revision
* Essential commands
* File management
* Process management
* Package management
* Networking basics
* Storage basics
* Service management

---

## Day 1 – Network Management

Topics:

* IP addressing
* DNS
* Routing
* Gateway configuration
* Hostname management
* Network troubleshooting

Labs:

* Configure static IP
* Configure DNS
* Add routes
* Troubleshoot connectivity issues

Scenarios:

* Server cannot reach internet
* DNS resolution failure
* Duplicate IP conflicts

---

## Day 2 – Storage & LVM

Topics:

* Partitioning
* Filesystems
* LVM architecture
* XFS
* EXT4

Labs:

* Create PV/VG/LV
* Extend storage
* Add new disks

Scenarios:

* Filesystem full
* Storage expansion request
* Disk failure investigation

---

## Day 3 – Boot Process & Systemd

Topics:

* BIOS
* UEFI
* GRUB
* Kernel
* Initramfs
* systemd

Labs:

* Create services
* Manage targets
* Analyze boot logs

Scenarios:

* Failed service startup
* Slow boot issue
* Boot failure investigation

---

## Day 4 – User Management

Topics:

* useradd
* usermod
* userdel
* passwd
* sudo

Labs:

* Create users
* Configure sudo access
* Account lock/unlock

Scenarios:

* New employee onboarding
* User offboarding
* Privilege escalation requests

---

## Day 5 – Permissions, ACL & UMASK

Topics:

* Permissions
* ACL
* UMASK
* SUID
* SGID
* Sticky Bit

Labs:

* Shared directory setup
* ACL configuration

Scenarios:

* Access denied issues
* Shared project folder management

---

## Day 6 – Firewall & SELinux

Topics:

* firewalld
* Zones
* Services
* SELinux contexts
* SELinux policies

Labs:

* Configure firewall rules
* Troubleshoot SELinux denials

Scenarios:

* Application blocked by firewall
* SELinux-related service failures

---

## Day 7 – NFS, AutoFS & Time Synchronization

Topics:

* NFS
* AutoFS
* Chrony
* NTP
* timedatectl

Labs:

* Configure NFS server/client
* Configure AutoFS
* Configure time synchronization

Scenarios:

* NFS mount failures
* Time drift problems

---

## Day 8 – Logging & Troubleshooting

Topics:

* journalctl
* rsyslog
* logrotate

Labs:

* Analyze logs
* Configure logging

Scenarios:

* Service crash investigation
* Disk full due to logs

---

## Day 9 – Performance Tuning & Patching

Topics:

* CPU analysis
* Memory analysis
* Disk performance
* Network performance

Tools:

* top
* vmstat
* iostat
* sar
* ss

Scenarios:

* High CPU usage
* Memory leaks
* Server slowdown

Additional:

* Enterprise patching process
* Security updates
* Kernel updates

---

## Day 10 – Disaster Recovery

Topics:

* Rescue Mode
* Emergency Mode
* Root password recovery
* Boot recovery

Labs:

* Reset root password
* Recover failed boot
* Fix broken fstab

Scenarios:

* Server inaccessible after update
* Corrupted boot configuration

---

# Additional Enterprise Topics

The workbook also includes:

## Linux Installation

* RHEL installation workflow
* Partition planning
* UEFI vs BIOS
* Kickstart
* PXE Boot

## Enterprise Patching

* Monthly patching cycle
* Emergency patching
* Validation procedures
* Rollback considerations

## Server Hardening

* SSH Hardening
* Password Policies
* SELinux
* Firewalld
* Auditd
* sudo Security

## Monitoring

* Nagios
* Zabbix
* Prometheus
* Grafana

## Backup & Recovery

* Backup types
* Restore validation
* Disaster recovery concepts

## Incident Management

* P1 incidents
* P2 incidents
* P3 incidents

---

# RHEL Version Comparison

Included:

| Feature               | RHEL 7  | RHEL 8   | RHEL 9   |
| --------------------- | ------- | -------- | -------- |
| Kernel                | ✓       | ✓        | ✓        |
| DNF                   |         | ✓        | ✓        |
| Podman                |         | ✓        | ✓        |
| Security Enhancements | Limited | Improved | Advanced |
| Lifecycle             | Legacy  | Current  | Latest   |

---

# Linux Distribution Comparison

Included comparisons:

* RHEL
* Rocky Linux
* AlmaLinux
* CentOS Stream
* Ubuntu
* Debian
* SUSE

Comparison areas:

* Package Management
* Stability
* Enterprise Adoption
* Commercial Support
* Use Cases

---

# Interview Preparation Section

Includes:

## Top 100 Linux Administration Scenarios

Each scenario contains:

```text
Situation
↓
Symptoms
↓
Investigation Path
↓
Tools To Use
↓
Possible Resolution Areas
```

No answer dumping.

Focus is on developing troubleshooting skills.

---

# Recommended Lab Environment

Option 1:

```text
Windows
 └── Hyper-V
      ├── RHEL VM
      ├── Rocky Linux VM
      └── Ubuntu VM
```

Option 2:

```text
Windows
 └── WSL2
      └── Linux Practice Environment
```

Option 3:

```text
VirtualBox
      ├── RHEL
      ├── AlmaLinux
      └── Ubuntu
```

---

# Recommended Resources

## Documentation

* Red Hat Documentation
* Rocky Linux Documentation
* AlmaLinux Documentation
* Ubuntu Documentation

## Learning Platforms

* Linux Journey
* LabEx
* Killercoda

## YouTube Channels

* Learn Linux TV
* TechWorld with Nana
* Red Hat

## Books

* RHCSA Cert Guide
* RHCE Cert Guide
* Linux Bible
* How Linux Works

---

# Repository Goal

By the end of this workbook, you should be able to:

* Perform common Linux administration tasks
* Troubleshoot production-like issues
* Understand enterprise patching workflows
* Explain technical decisions during interviews
* Handle scenario-based Linux interview questions confidently

---

## Disclaimer

This repository is intended for educational purposes and interview preparation. Always validate commands and procedures in a lab environment before applying them to production systems.

 
