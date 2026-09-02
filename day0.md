# Chapter 0 — Syllabus & Quick Command Revision (Advanced Edition)

## How to use this chapter

This chapter is no longer a "look up the flag" cheat-sheet — treat it as a **standalone 2–3 hour deep-revision session**. For every domain below you'll get:

1. **Command syntax** (the base layer you already know)
2. **Advanced flags & lesser-used options** that separate "I've run this once" from "I run this in production"
3. **Output interpretation** — what the numbers/columns actually mean and what "bad" looks like
4. **Log analysis** — where this subsystem logs to, and how to read it under pressure
5. **A "why a product company cares" note** — the reasoning an interviewer is actually listening for

If a section takes you 10 minutes, you skimmed it. Go back and actually run the diagnostic commands against a live VM while reading.

---

## High-Level Syllabus (10-Day Roadmap)

| Day | Focus | Core Topics |
|---|---|---|
| 1 | Storage Foundations | LVM (PV/VG/LV), extending/shrinking, snapshots, resilience |
| 2 | Networking | nmcli/nmtui, bonding/teaming, connectivity troubleshooting |
| 3 | Systemd & Service Management | Units, targets, dependencies, masking, custom services, journalctl |
| 4 | Identity & Access | User management, password aging, sudo, sudoers, /etc/skel |
| 5 | Permissions Deep Dive | UMASK, ACLs (setfacl/getfacl), when Unix permissions aren't enough |
| 6 | Firewalld & Network Security | Zones, rich rules, port management, masquerading |
| 7 | SELinux | Modes, contexts, booleans, denial troubleshooting (sealert/audit2allow) |
| 8 | Shared Storage & Time | AutoFS/NFS, stale mount troubleshooting, Chrony/NTP, timedatectl |
| 9 | Recovery & Automation Bonus | Password/root recovery, cron/at, logrotate, shell scripting basics, sysctl |
| 10 | Performance & Package Management | tuned-adm, bottleneck triage (CPU/mem/disk/net), dnf/rpm/flatpak |

Additional chapters (A–E) on installation/patching, hardening, distro comparisons, cloud/SRE bridging, and study resources sit outside the 10-day core and can be read in any order once you've finished Day 10.

---

## 1. Storage Commands

### LVM — beyond the basics

```bash
pvcreate /dev/sdb /dev/sdc
pvs -o +pv_used                        # Show how much of each PV is actually consumed
pvdisplay -m                           # -m shows the PE-to-LE mapping (which physical extents back which LV)

vgcreate -s 32M vg_data /dev/sdb /dev/sdc   # -s sets PE size explicitly (default 4M; matters for very large VGs)
vgs -o +vg_free_count,vg_extent_count
vgextend vg_data /dev/sdd
vgreduce vg_data /dev/sdd              # Remove a PV from a VG (must be empty first — pvmove off it)
vgchange -ay vg_data                   # Activate a VG (needed after import on a new host)

lvcreate -n lv_app -L 10G vg_data
lvcreate -n lv_app -l 100%FREE vg_data
lvcreate -i2 -I64 -n lv_striped -L 20G vg_data   # Striped LV across 2 PVs, 64K stripe size — for throughput
lvcreate -m1 -n lv_mirrored -L 10G vg_data       # LVM mirroring (rare in prod — most use RAID/SAN redundancy instead)
lvextend -r -L +5G /dev/vg_data/lv_app           # -r resizes the filesystem in the same command (XFS/ext4 only)
lvreduce -L -5G /dev/vg_data/lv_app              # Never works on a mounted XFS volume — XFS cannot shrink, period
lvs -o +devices                        # Shows which PVs actually back this LV — critical when troubleshooting a failing disk
```

**Snapshots — the part people gloss over:**
```bash
lvcreate -s -n lv_app_snap -L 2G vg_data/lv_app
```
A snapshot's `-L` size isn't "the size of a copy" — it's **COW (copy-on-write) overflow space**. Every time a block in the *origin* LV changes, the *old* block gets copied into the snapshot first. If your snapshot fills up before you remove it, LVM automatically **invalidates** the snapshot (it becomes useless) — this is one of the most common "why did my backup snapshot die" incidents. Size snapshots based on expected *write volume* during the snapshot's lifetime, not the volume's total size.

```bash
lvs -o +snap_percent                   # Watch snapshot fill percentage — alert before it hits 80-90%
lvconvert --merge vg_data/lv_app_snap  # Rollback: merges snapshot back into origin (requires reboot if origin is root/in-use)
```

**Why a product company cares:** They're not testing whether you memorized `lvcreate`. They're testing whether you understand *COW semantics*, because a snapshot that silently invalidates during a maintenance window is a real 2 AM incident.

### Partitioning
```bash
lsblk -f                               # -f adds filesystem type + UUID + mountpoint in one view
parted /dev/sdb print                  # Non-interactive print of partition table
parted -s /dev/sdb mklabel gpt mkpart primary xfs 0% 100%   # Scripted GPT partitioning
partprobe                              # Reload partition table without reboot — but note some kernels still need `udevadm settle` after
udevadm settle                         # Wait for udev to finish processing device events (races are real with fresh disks)
```

### Filesystems — deeper than mkfs
```bash
mkfs.xfs -f /dev/vg_data/lv_app        # -f force (needed if FS signature already exists)
mkfs.xfs -i size=512 /dev/vg_data/lv_app   # Larger inode size — needed if you plan heavy xattr/ACL usage
xfs_info /mount/point                  # Show block size, agcount, inode size — useful before deciding to grow
xfs_growfs /mount/point                # Grow XFS — again, NO shrink support, ever, by design
resize2fs /dev/vg_data/lv_app          # ext4 can grow AND shrink (unlike XFS)
xfs_repair -n /dev/vg_data/lv_app      # -n = dry run, check without repairing (always do this first)
fsck -y /dev/vg_data/lv_app            # -y auto-answer yes to all repair prompts (use with caution)
tune2fs -l /dev/vg_data/lv_app         # ext4: dump superblock info (last check time, mount count, etc.)
xfs_db -c "sb 0" -c "print" /dev/vg_data/lv_app   # Advanced: inspect XFS superblock directly (rarely needed, but interviewers love asking if you know it exists)
```

### Mount / fstab — the failure modes that matter
```bash
mount /dev/vg_data/lv_app /data
mount -o remount,ro /data              # Remount read-only without unmounting — used to freeze writes during investigation
umount /data
blkid                                  # Always prefer UUID= over /dev/sdX in fstab — device names shift across reboots
echo "UUID=xxxx /data xfs defaults,nofail 0 0" >> /etc/fstab
```
`nofail` is the option interviewers specifically probe for: without it, a missing/failed disk at boot can drop the **entire server** into emergency mode waiting for that one mount. `nofail` lets boot continue and the mount simply fails silently (log it and alert instead).

```bash
mount -a                               # Validates fstab syntax without rebooting — run this after EVERY fstab edit, no exceptions
findmnt --verify                       # Explicitly validates fstab entries and reports errors
findmnt -D                             # Disk usage view of currently mounted filesystems
```

### Disk Usage — reading beyond `df -h`
```bash
df -hT
df -i                                  # Inode usage — "No space left on device" with df showing free space = you're out of inodes, not blocks
du -sh /path/*
du -ah /path | sort -rh | head -20
lsof +L1                               # Find files that are DELETED but still held open by a process (classic "disk full but du shows free space" cause)
lsof /data                             # Which processes have files open on this mount (needed before you can unmount)
```

### Storage Log Analysis
- **Kernel/disk errors**: `journalctl -k | grep -iE "error|fail|i/o error"` — I/O errors from a failing disk surface here first, usually before SMART data confirms it.
- **XFS-specific corruption events**: search `journalctl -k` or `/var/log/messages` for `XFS (dm-N):` entries — these indicate filesystem-level corruption, not just device I/O errors.
- **LVM activity**: `/var/log/messages` (or journalctl) logs PV/VG/LV creation, activation, and failures under the `lvm` tag.
- **SMART health** (the step before it becomes a kernel I/O error): `smartctl -a /dev/sdb` — look at `Reallocated_Sector_Ct` and `Current_Pending_Sector`; non-zero and climbing = disk is dying, replace proactively.

---

## 2. Network Commands

### nmcli / nmtui — production patterns, not just "set an IP"
```bash
nmcli connection show
nmcli connection show eth0             # Full detail dump of one profile — everything nmcli knows about it
nmcli device status
nmcli con up "eth0"
nmcli con mod eth0 ipv4.addresses 192.168.1.10/24 ipv4.gateway 192.168.1.1 ipv4.method manual
nmcli con mod eth0 ipv4.dns "8.8.8.8 1.1.1.1"
nmcli con mod eth0 +ipv4.addresses 192.168.1.11/24   # Add a SECOND IP to the same interface without removing the first
nmcli con mod eth0 connection.autoconnect yes
nmcli con reload                       # Reload connection profiles from disk after manual file edits in /etc/NetworkManager/system-connections/
nmtui
```

### Bonding/Teaming — know the modes, not just the command
```bash
nmcli con add type bond ifname bond0 mode active-backup
nmcli con add type ethernet ifname eth0 master bond0
nmcli con add type ethernet ifname eth1 master bond0
cat /proc/net/bonding/bond0            # Check "MII Status: up" for each slave and "Currently Active Slave"
```
**Bonding modes worth knowing cold** (interviewers ask "why would you pick mode X over Y"):
- `active-backup` (mode 1) — one active NIC, one standby, no switch config needed. Safest default.
- `802.3ad` / LACP (mode 4) — true link aggregation, needs switch-side LACP config, gives real throughput increase.
- `balance-rr` (mode 0) — round-robin, rarely used in prod (out-of-order packet issues).

### Connectivity Troubleshooting — the actual escalation order
```bash
ip addr show                           # 1. Do I even have an IP?
ip route show                          # 2. Do I have a route to get there?
ip route get 8.8.8.8                   # Shows exactly which interface/gateway will be used for a destination
ping -c4 <gateway>                     # 3. Can I reach the gateway (L2/L3 local)?
ping -c4 8.8.8.8                       # 4. Can I reach the internet by IP (rules out DNS)?
dig google.com                         # 5. Is DNS resolving?
curl -v telnet://host:port             # 6. Is the specific PORT reachable (app-layer)?
nc -zv host port
ss -tulnp                              # 7. Is something actually listening locally on the port I expect?
ss -antp | grep ESTABLISHED            # Active connections — useful for spotting connection leaks/exhaustion
tcpdump -i eth0 port 443 -nn           # 8. Packet-level: is traffic even arriving/leaving the NIC?
mtr host                               # Combined ping+traceroute with live loss % per hop — better than traceroute for intermittent issues
```
This 8-step order (local IP → route → L3 reachability → DNS → port → local listener → active connections → packet capture) **is the actual answer** to "walk me through troubleshooting a connectivity issue" — memorize the *order*, not just the commands.

### Network Log Analysis
- `journalctl -u NetworkManager` — connection state changes, DHCP lease events, profile activation failures.
- `journalctl -k | grep -i "link is"` — physical link up/down events straight from the kernel driver.
- `/var/log/messages` — DHCP client (`dhclient`) lease acquisition/renewal failures.
- `ss -s` — socket summary statistics; a growing `TIME-WAIT` count under load can indicate connection churn or an app not closing sockets properly.

---

## 3. User & Group Management Commands

```bash
useradd -m -s /bin/bash -c "Full Name" username
useradd -m -G wheel,developers -e 2026-12-31 username   # -e sets account EXPIRY date — common for contractor accounts
usermod -aG groupname username         # -a = APPEND. Forgetting -a wipes all other supplementary groups — a classic production mistake
usermod -L username                    # Lock (prepends ! to password hash — SSH keys still work unless separately disabled)
usermod -s /sbin/nologin username
userdel -r username                    # -r removes home dir + mail spool too

groupadd -g 5000 groupname             # Explicit GID — needed for consistency across NFS/AD-integrated environments
gpasswd -a username groupname
gpasswd -d username groupname          # Remove user from a group (alternative to editing /etc/group by hand)

passwd -S username                     # Quick status: locked/unlocked, password aging state, last change date
chage -l username
chage -M 90 -m 7 -W 7 -I 14 username   # Max age 90, min age 7 (can't change more than once/week), warn 7 days before, inactive 14 days after expiry before lock
```

**Key files, and what actually breaks when they're wrong:**
- `/etc/passwd` — world-readable; a malformed line (wrong field count) can break `getent passwd` and cascade into login failures for *everyone*, not just one user.
- `/etc/shadow` — must stay `0000` or `0400` owned by root; a permissions mistake here is a hard security finding in any audit.
- `/etc/login.defs` — defines default `UID_MIN`/`UID_MAX`, password aging defaults applied to *new* users only (doesn't retroactively change existing accounts — a common interview trap question).
- `/etc/skel/` — template copied into new home dirs; anything you drop here (`.bashrc` snippets, MOTD, default umask) becomes org-wide policy for all future accounts.

### sudo / sudoers — the part product companies actually probe
```bash
visudo                                 # ALWAYS use this, never edit /etc/sudoers directly — it syntax-checks before saving
visudo -c                              # Check syntax of existing file without opening editor
```
```
# /etc/sudoers.d/app-team   (preferred over editing the main file — drop-in files survive updates cleanly)
%developers ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart app.service
```
Interviewers listen for: do you know that `%group` syntax targets a *group*, that NOPASSWD should be scoped to *specific commands* (never blanket `NOPASSWD: ALL` in prod), and that `/etc/sudoers.d/` drop-ins are the maintainable pattern over one giant sudoers file.

### User/Auth Log Analysis
```bash
journalctl -u sshd                     # SSH auth attempts, key vs password auth, disconnect reasons
grep "Failed password" /var/log/secure | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn   # Top source IPs for failed SSH logins
lastb                                  # Failed login attempts (bad logins)
last                                   # Successful login history
faillock --user username               # PAM-based failed-attempt lockout status (RHEL 8+, replaces pam_tally2)
journalctl -u sudo                     # sudo invocation audit trail — who ran what, as whom
```
`/var/log/secure` is the single most-referenced log file in a "someone's account got compromised, what do you check" interview scenario — know it by name, not just "some auth log somewhere."

---

## 4. Permissions & ACL Commands

```bash
chmod 750 file
chmod u+x,g-w,o-rwx file
chmod -R u+rX,go-w /dir                # Capital -X = execute only if it's a directory or already executable for someone — avoids making every FILE executable during a recursive chmod
chown user:group file
chown -R user:group /dir
chmod 2770 /dir                        # SGID on a directory — new files inherit the directory's GROUP, not the creator's primary group (huge for shared team directories)
chmod 1777 /tmp                        # Sticky bit — users can only delete their OWN files even with directory write access
```

```bash
umask                                  
umask 022                              # Session-only
```
Persistent umask locations, in the order they actually take effect:
- `/etc/profile` / `/etc/bashrc` — system-wide default for interactive shells
- `~/.bashrc` / `~/.bash_profile` — per-user override
- `/etc/login.defs` (`UMASK` directive) — affects `useradd`-created home dir defaults specifically

```bash
setfacl -m u:username:rwx file
setfacl -m g:groupname:rx file
setfacl -m d:u:username:rwx /dir       # Default ACL — inherited automatically by every NEW file/subdir created inside
getfacl file                           # Note the "+" after normal permission bits in `ls -l` output — that's your visual cue an ACL exists
setfacl -x u:username file
setfacl -b file                        # Strip all ACLs — restores to plain Unix permission model
```
**When plain permissions genuinely aren't enough:** the classic case is a shared project directory where you need User A to have read-write, User B to have read-only, and everyone else no access — but A and B aren't in the same group and you can't/won't restructure your group hierarchy for one folder. That's a textbook ACL interview scenario.

### Permissions Log Analysis
Permission denials rarely log anywhere by default (a plain `EACCES` is silent). This is itself a good interview point: **if a permission-based denial isn't showing in application logs, the next place to check is SELinux (`ausearch -m avc`), not the permission bits themselves** — because SELinux denials get logged, but a plain DAC (discretionary access control) denial usually doesn't, beyond an app's own error output.

---

## 5. SELinux Commands

```bash
getenforce
setenforce 0                           # Permissive — logs would-be denials but allows the action (temporary, resets on reboot)
setenforce 1                           # Enforcing
# Persistent: edit SELINUX= line in /etc/selinux/config, requires reboot to take effect
```

```bash
ls -Z file                             # user:role:type:level — the TYPE is what actually drives most access decisions
ps -Z                                  
chcon -t httpd_sys_content_t file      # Temporary — reverts on next restorecon or relabel
restorecon -Rv /path                   # Restore to policy-defined default context
semanage fcontext -a -t httpd_sys_content_t "/web(/.*)?"   # PERSISTENT context rule, survives restorecon and relabels
restorecon -Rv /web                    # Apply the new persistent rule immediately
semanage fcontext -l | grep web        # List custom context rules you've defined
```
The interview trap here: `chcon` changes are **lost** the next time the system relabels or someone runs `restorecon`. `semanage fcontext` writes the rule into policy so it survives — if your answer to "how do I permanently fix a context" is `chcon`, that's the wrong answer.

```bash
getsebool -a | grep httpd
setsebool httpd_can_network_connect on          # Runtime only
setsebool -P httpd_can_network_connect on       # -P = persistent across reboot
```

**Denial troubleshooting — the actual workflow:**
```bash
ausearch -m avc -ts recent             # Raw audit denials, most recent
sealert -a /var/log/audit/audit.log    # Human-readable explanation + suggested fix — start here, not with raw audit.log
audit2allow -a                         # Show what a NEW policy module would look like, based on recent denials
audit2allow -a -M mymodule && semodule -i mymodule.pp   # Generate and load a custom policy module
```
**Why this matters to a product company:** `setenforce 0` "fixing" a denial is the single most common wrong-answer red flag in SELinux interview questions — it disables enforcement system-wide instead of fixing the specific context/boolean issue. The correct workflow is always: identify via `sealert`/`ausearch` → fix via `restorecon`/`semanage fcontext`/`setsebool -P` → only use `audit2allow` custom modules as a last resort, because overly broad custom policy modules are themselves a security smell.

---

## 6. Systemd / systemctl / journalctl Commands

```bash
systemctl start/stop/restart/reload service
systemctl status service                # -l for full (unwrapped) log lines, --no-pager for scripting
systemctl enable/disable service
systemctl enable --now service
systemctl is-active / is-enabled / is-failed service
systemctl mask service                  # Symlinks the unit to /dev/null — CANNOT be started even manually, stronger than disable
systemctl unmask service
systemctl daemon-reload                 # Mandatory after editing/adding any unit file, or systemd silently keeps using the old definition
systemctl list-units --type=service --state=failed   # Fast way to see what's currently broken across the whole box
systemctl list-dependencies service --all
systemctl show service -p Restart -p RestartSec      # Inspect specific unit properties without a full status dump
systemctl edit service                  # Creates a drop-in override file — the correct way to customize a vendor unit, never edit the vendor's .service file directly
systemctl get-default / set-default multi-user.target
```

**Custom unit essentials** (`/etc/systemd/system/myapp.service`):
```ini
[Unit]
Description=My App
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/bin/myapp
Restart=on-failure
RestartSec=5
User=appuser

[Install]
WantedBy=multi-user.target
```
`After=` vs `Wants=`/`Requires=` is a frequent interview question: `After=` is purely ordering (doesn't pull the dependency in), `Wants=` pulls it in but tolerates its failure, `Requires=` pulls it in and fails this unit too if the dependency fails.

### journalctl — this is where "2-3 hours" actually lives
```bash
journalctl -u service_name
journalctl -u service_name --since "1 hour ago"
journalctl -u service_name --since "2026-09-01 10:00:00" --until "2026-09-01 11:00:00"
journalctl -p err                       # Priority filter: emerg/alert/crit/err/warning/notice/info/debug
journalctl -p err..alert                # Range of priorities
journalctl -b                           # Current boot only
journalctl -b -1                        # Previous boot — essential after an unexpected reboot/crash
journalctl --list-boots                 # See all recorded boot sessions with their IDs
journalctl -k                           # Kernel messages only (replaces dmesg for persistent logs)
journalctl -f                           # Follow, like tail -f
journalctl -o json-pretty -u sshd       # Structured output — useful when piping into scripts/log shippers
journalctl _PID=1234                    # Filter by exact PID
journalctl -g "OOM|out of memory"       # Grep within journal directly (regex support)
journalctl --disk-usage
journalctl --vacuum-size=500M           # Cap total journal size
journalctl --vacuum-time=2weeks         # Cap by age instead of size
```
**Persistent logging setup** (default is often volatile/RAM-only on minimal installs):
```bash
mkdir -p /var/log/journal
systemd-tmpfiles --create --prefix /var/log/journal
systemctl restart systemd-journald
```

**Correlating logs across a reboot — a real scenario:** a service crashed the box. The workflow is: `journalctl --list-boots` to find the boot ID right before the crash → `journalctl -b -1 -p err` to see errors right up to the moment it died → cross-reference the last kernel lines (`journalctl -b -1 -k | tail -50`) to check if it was an OOM kill, a kernel panic, or a clean shutdown that just didn't come back up.

---

## 7. Firewall Commands

```bash
firewall-cmd --state
firewall-cmd --get-active-zones
firewall-cmd --get-default-zone
firewall-cmd --zone=public --add-service=http --permanent
firewall-cmd --zone=public --add-port=8080/tcp --permanent
firewall-cmd --reload                   # Applies --permanent changes to the runtime config
firewall-cmd --zone=public --list-all
firewall-cmd --zone=public --list-all --permanent    # Compare against the ABOVE (runtime) — mismatches mean someone made an un-reloaded or un-persisted change
```

```bash
firewall-cmd --zone=public --add-rich-rule='rule family="ipv4" source address="192.168.1.0/24" service name="ssh" accept' --permanent
firewall-cmd --zone=public --add-rich-rule='rule family="ipv4" source address="10.0.0.5" port port="3306" protocol="tcp" reject' --permanent
firewall-cmd --zone=public --list-rich-rules
```

```bash
firewall-cmd --zone=public --add-masquerade --permanent   # NAT — required on any box acting as a gateway/router for other hosts
firewall-cmd --zone=public --query-masquerade
```

**Zones — the conceptual model interviewers actually probe:** firewalld zones represent *trust levels* for network interfaces, not just rule containers. `public` (default, untrusted) vs `internal`/`trusted` (higher trust, more permissive) vs `drop` (silently discards everything). The real interview question is "why would you assign different NICs to different zones on a multi-homed server" — e.g., the public-facing NIC in `public`, the internal management NIC in `internal`.

```bash
nft list ruleset                        # firewalld is a frontend; nftables is the actual kernel-level backend since RHEL 8
```

### Firewall Log Analysis
```bash
firewall-cmd --zone=public --add-rich-rule='rule family="ipv4" source address="1.2.3.4" service name="ssh" log prefix="ssh-block" level="info" limit value="3/m" accept' --permanent
journalctl -k | grep "ssh-block"        # Logged firewall matches surface via the kernel log, tagged by your chosen prefix
```

---

## 8. Package Management Commands

```bash
dnf install pkg
dnf remove pkg
dnf update / dnf upgrade                # update = all except kernel by some configs, upgrade = alias for update in modern dnf, but always confirm with dnf config
dnf search keyword
dnf history                             # Full transaction log
dnf history info <id>                   # Exactly what a specific transaction did (packages added/removed/upgraded)
dnf history undo <id>                   # Rollback a transaction
dnf repolist
dnf repolist --all                      # Include disabled repos
dnf provides "*/bin/somecommand"        # "Which package do I need to install to get this binary/file"
dnf makecache                           # Force-refresh repo metadata (useful when a new package should exist but dnf can't see it yet)
dnf list installed | grep pkg
dnf check-update                        # Dry-run: what WOULD be updated, without applying anything
```

```bash
rpm -qa | grep pkg
rpm -qi pkg
rpm -ql pkg
rpm -qf /path/to/file                   # Reverse lookup: which package owns this file (essential during incident triage — "who put this binary here")
rpm -V pkg                              # VERIFY — compares installed files against the package's recorded checksums/perms; flags tampering or accidental modification
rpm --import GPG-KEY-FILE               # Import a vendor's signing key before installing their signed packages
```

```bash
flatpak install remote appname
flatpak list
flatpak update
```
**Where flatpak actually fits vs dnf/rpm:** flatpak is for sandboxed desktop/GUI applications with their own bundled runtime — essentially irrelevant on a headless production RHEL server. Knowing to say "we wouldn't use this in a server context, dnf/rpm covers system packages" is itself the correct interview answer, not a deep-dive into flatpak internals.

---

## 9. Time Sync Commands

```bash
timedatectl status
timedatectl set-timezone Asia/Kolkata
timedatectl set-ntp true
timedatectl list-timezones | grep Kolkata
```

```bash
chronyc sources -v                      # NTP source list with stratum, reachability, and offset — the "-v" adds column explanations
chronyc tracking                        # Current sync accuracy: System time offset, RMS offset, Frequency drift
chronyc sourcestats                     # Longer-term jitter/drift stats per source — useful for diagnosing "keeps drifting" issues
systemctl status chronyd
```

**Why time sync is a bigger interview topic than it looks:** clock drift breaks Kerberos/AD auth (tickets have tight time-window tolerances), causes false-positive/negative results in cron-scheduled jobs, produces misleading log timestamps during incident correlation across multiple hosts, and can silently break TLS certificate validation (`not yet valid`/`expired` errors) if drift is large enough. "What breaks when NTP is down" is a real question, and the answer is "auth, cron, logs, and TLS" — not just "the clock is wrong."

---

## 10. Automount/NFS Commands

```bash
# Server side
cat /etc/exports
exportfs -ra                            # Re-export everything defined in /etc/exports
exportfs -v                             # Show currently active exports with their options
showmount -e server_ip                  # Client-side: what does this server actually export right now

# Client side — direct mount
mount -t nfs -o vers=4.2,rsize=32768,wsize=32768 server:/share /mnt/point
```

**AutoFS — the part people memorize wrong:**
```bash
cat /etc/auto.master
# /misc /etc/auto.misc
cat /etc/auto.misc
# data  -rw,soft,intr   server:/export/data
systemctl restart autofs
automount -v                            # Verbose foreground run — invaluable for debugging map file syntax errors
```
The `soft` vs `hard` NFS mount option is a genuine interview differentiator: `hard` (default) retries forever if the server is unreachable, which can hang processes indefinitely (safer for data integrity, worse for availability); `soft` gives up after a timeout and returns an I/O error to the application (better availability, risk of silent data loss/corruption on a write that "times out" mid-flight). Product companies ask this because picking wrong has actually caused real outages.

**Stale mount troubleshooting:**
```bash
umount -l /mnt/point                    # Lazy unmount — detaches from the namespace immediately, cleans up once nothing references it anymore
umount -f /mnt/point                    # Force — for when the server is genuinely gone and hard is hanging things
showmount -e server_ip                  # Confirm server is even still exporting the share
rpcinfo -p server_ip                    # Check NFS/rpcbind services are actually running on the server side
```

### NFS/AutoFS Log Analysis
```bash
journalctl -u nfs-server                # Server-side export/service events
journalctl -u autofs                    # Mount/unmount trigger events, map parsing errors
dmesg | grep -i nfs                     # Kernel-level NFS timeout/retry messages — this is where "server not responding" first appears
```

---

## 11. Performance/Tuning Commands

```bash
tuned-adm list
tuned-adm active
tuned-adm profile throughput-performance
tuned-adm recommend                     # Shows what profile tuned WOULD recommend for this hardware/workload
```

### Reading the actual numbers, not just running the command

```bash
top / htop
vmstat 2 5
```
In `vmstat` output, the columns that actually matter for triage: **`r`** (runnable processes waiting for CPU — sustained values higher than your core count = CPU-bound), **`b`** (processes blocked on I/O), **`si`/`so`** (swap in/out — any non-zero sustained value means you're memory-constrained and thrashing), **`wa`** (in `top`, % CPU time waiting on I/O — high `wa` with low actual CPU use points at disk, not CPU).

```bash
iostat -xz 2
```
Key columns: **`%util`** (device busy percentage — sustained near 100% means the disk is your bottleneck), **`await`** (average I/O wait time in ms — compare against `svctm` if present; a growing gap between them means queueing, not raw device slowness), **`avgqu-sz`** (average queue depth — high values confirm requests are backing up).

```bash
sar -u 1 5                              # CPU utilization
sar -r 1 5                              # Memory utilization
sar -d 1 5                              # Disk I/O
sar -n DEV 1 5                          # Network throughput per interface
sar -f /var/log/sa/sa15                 # Read HISTORICAL data from a specific day's archived sar file — this is what makes sar different from top/vmstat: it's the only tool here with automatic historical retention
```
`sar`'s real interview value is answering **"the incident happened at 3 AM and nobody was watching — how do you investigate after the fact"** — `top`/`vmstat`/`iostat` are live-only; `sar` (via `sysstat`'s cron-collected archives in `/var/log/sa/`) is the one tool in this list that still has data the next morning.

```bash
free -h                                 # Note: "available" (not "free") is the real number that matters — it accounts for reclaimable cache/buffers
mpstat -P ALL 2                         # Per-CPU core breakdown — reveals a single hot core (bad thread affinity/single-threaded bottleneck) that an aggregate view would hide
pidstat -u 2                            # Per-PROCESS CPU usage over time — narrows "system is slow" down to "this specific PID is slow"
pidstat -d 2                            # Per-process disk I/O — find which process is actually hammering the disk
```

**The escalation order that actually answers "server is slow, what do you check":**
1. `top`/`htop` — is it CPU, memory, or load average that's abnormal, at a glance
2. `vmstat` — confirm CPU run-queue vs I/O-wait vs swap activity
3. `iostat -xz` — if I/O-bound, which device, how saturated
4. `pidstat` — which specific process is responsible
5. `sar` — was this a spike or has it been building for hours/days (historical context changes the whole diagnosis)

---

Once this chapter feels genuinely familiar — not memorized, but familiar enough that you could reconstruct the escalation orders and COW/context/zone concepts from memory — move to **Day 1 — LVM & Storage Fundamentals**.
