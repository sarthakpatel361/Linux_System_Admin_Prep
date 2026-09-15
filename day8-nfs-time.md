# Day 8 — Shared Storage & Time (Full Study Edition, with Answers)

**Estimated time: 5–6 hours**

Today covers two topics that seem unrelated but share a common theme: both fail in ways that look like something else entirely if you don't know exactly where to look. An NFS client hanging looks like a network problem, a permissions problem, or an application bug depending on which symptom you happen to notice first. A cluster with drifted clocks produces authentication failures, broken cron scheduling, and confusing log correlation — none of which look like "the clock is wrong" on the surface. This day builds the specific diagnostic instincts for both: NFS exports, AutoFS automounting, the full stale-mount troubleshooting workflow, Chrony/time-sync, and — as the advanced capstone — NFSv4's pseudo-filesystem model and Kerberized NFS, which is where storage and identity genuinely intersect.

---

## Topic 1: NFS Exports & Server-Side Configuration

### Quick Review
- `/etc/exports` defines what's shared and with what options; `exportfs -ra` re-reads it without a full service restart.
- **`root_squash`** (default) maps the remote root user to an unprivileged local user (`nobody`) — a critical security default, not an inconvenience to disable casually.
- NFSv4 uses a single **pseudo-filesystem root** (`fsid=0` or equivalent) rather than exporting each share as an independent mount point the way NFSv3 does.
- `showmount -e` (client-side) and `exportfs -v` (server-side) are your two views of the same export state — use both when diagnosing a mismatch.

### Quick Learning

NFS exports look simple on the surface — a path and a client spec in `/etc/exports` — but the option flags are where the actual security and correctness decisions live, and `root_squash` specifically is the one interviewers use to test whether you understand WHY it exists rather than just what it does. Without it, a user with root access on ANY client machine that can mount the share would have root-equivalent access to the exported filesystem on the SERVER — collapsing the trust boundary between client and server entirely, since root on a client is often far less trustworthy than root on the server itself (a client could be a laptop, a dev VM, anything).

**root_squash — the trust boundary NFS exports are built around:**
```
  /etc/exports:  /data  10.0.0.0/24(rw,sync,root_squash)

  Client (10.0.0.50) user runs as root, accesses /data
        │
        ▼
  WITHOUT root_squash:                    WITH root_squash (default):
  ─────────────────────                    ──────────────────────────
  Server treats this request AS ROOT       Server maps remote root (UID 0)
  ⇒ full read/write/delete access to        to a LOCAL unprivileged UID
    EVERYTHING on the export, same          (typically "nobody", UID 65534)
    as if logged in locally as root         ⇒ remote "root" now has only
                                               whatever ordinary-user access
  Any client's root user = server's         a regular user would have on
  root user, for this export — a            this export, NOT full server-
  massive trust assumption most             root-equivalent access
  environments should never make

  no_root_squash exists for specific, narrow, deliberate cases
  (e.g., a diskless boot server needing genuine root operations on
  the export) — it should never be a casual, default choice.
```

### Implementation (Learn by Applying)

**Scenario:** Export a shared data directory to a specific subnet with correct, production-appropriate options, then deliberately demonstrate WHY `root_squash` matters by comparing behavior with and without it.

```bash
mkdir -p /data/shared
chown nobody:nobody /data/shared   # Since root will be squashed TO this user, ownership matters for the intended access model
chmod 755 /data/shared

cat >> /etc/exports << 'EOF'
/data/shared  10.0.0.0/24(rw,sync,root_squash)
EOF

exportfs -ra
exportfs -v                          # Confirm the export is active with the expected options
systemctl enable --now nfs-server
```

From a client, confirm the export is visible and mount it:
```bash
showmount -e nfsserver
mount -t nfs nfsserver:/data/shared /mnt
```

Demonstrate root_squash's actual effect:
```bash
# On the CLIENT, as root, attempt to create a file owned by root
touch /mnt/root_owned_file.txt
ls -l /mnt/root_owned_file.txt      # Owner shows as "nobody", NOT root — root_squash mapped the request
```

Compare against the risk `no_root_squash` introduces (understand, don't deploy this casually):
```bash
sed -i 's/root_squash/no_root_squash/' /etc/exports
exportfs -ra
# Remount on the client and repeat the test
umount /mnt && mount -t nfs nfsserver:/data/shared /mnt
touch /mnt/root_owned_file2.txt
ls -l /mnt/root_owned_file2.txt     # NOW owned by actual root — full server-root-equivalent access from any client root user
# Revert immediately - this configuration should not persist
sed -i 's/no_root_squash/root_squash/' /etc/exports
exportfs -ra
```

### Interview Questions — with Answers

**1. Explain exactly what `root_squash` does and why it's the default behavior rather than something administrators have to opt into.**

`root_squash` remaps any request arriving with UID 0 (root) from a client onto an unprivileged local user (typically `nobody`) on the NFS server, meaning a client's root user gets only ordinary-user-level access to the export rather than genuine root-equivalent access to the server's filesystem. It's the default specifically because NFS's trust model would otherwise be dangerously permissive by default — any client machine that can mount an export effectively grants its root user (which could be a compromised or less-trusted machine, like a developer's laptop or VM) full root access to the exported data on the server, collapsing a trust boundary that should exist between "who has root on some random client" and "who has root-level access to our shared server storage" — making the secure behavior the default, requiring explicit opt-out (`no_root_squash`) for the narrow cases that genuinely need it, is the correct security posture.

**2. Describe a genuinely legitimate scenario where `no_root_squash` would be the correct, deliberate choice, despite its risks.**

A common legitimate case is a diskless boot / PXE-boot server, where client machines boot their entire root filesystem over NFS and genuinely need root-level operations (creating device nodes, setting specific ownership, performing operations only root can do) against that NFS-mounted root filesystem as part of normal, expected boot and system operation — squashing root in that scenario would break the fundamental use case, since the client's OS genuinely needs to operate as root against its own root filesystem, which happens to be NFS-mounted. Another scenario: certain backup/restore tools that specifically need to preserve exact ownership/permissions including root-owned files, run from a trusted, tightly-controlled backup server. In both cases, the key is that the CLIENT side is itself highly trusted and controlled — `no_root_squash` should never be applied broadly to an export reachable by arbitrary or lightly-trusted client machines.

**3. What's the difference between `exportfs -ra` and restarting the `nfs-server` service entirely — when would you need the more disruptive full restart?**

`exportfs -ra` re-reads `/etc/exports` and re-exports (re-applies) everything defined there, WITHOUT restarting the underlying NFS server daemons or disrupting currently-connected clients' existing mounts — this is the correct, low-disruption way to apply changes after editing `/etc/exports` in the vast majority of cases. A full `systemctl restart nfs-server` is more disruptive, briefly interrupting the NFS service entirely (and by extension, potentially causing currently-connected clients to experience a momentary NFS service interruption, which behaves differently depending on whether they're mounted `soft` or `hard`, per Day 1/2's earlier NFS mount option coverage) — I'd reserve a full restart for situations where the NFS server process itself is genuinely misbehaving or needs to pick up a configuration change that `exportfs -ra` alone doesn't cover (certain deeper NFS daemon-level settings, not just the exports list itself).

**4. A client reports they can mount an NFS export successfully, but every file operation returns "permission denied," even for files that should be world-readable at the Unix level. Given today's topic, what would you check first?**

Given `root_squash`'s behavior specifically, I'd check whether the client is operating as root and whether the export has `root_squash` in effect — if the export's underlying files/directories are owned in a way that assumes root-level access (e.g., files owned specifically by `root:root` with restrictive permissions, intended to be manipulated by an admin who'd normally have root access), squashing that client's root to `nobody` means the request is now evaluated as an ORDINARY user against those root-owned, restrictively-permissioned files — producing exactly this "permission denied despite seeming like it should work" symptom, since the actual effective identity making the request isn't what the client believes it is. I'd confirm by checking `exportfs -v` for the export's options and comparing the target files' ownership/permissions against what an unprivileged `nobody`-mapped identity would actually be allowed to do.

**5. Compare `showmount -e` (run from a client) against `exportfs -v` (run on the server) — why would you need both when diagnosing an export visibility issue, rather than trusting just one?**

`exportfs -v` on the server shows what the NFS server ITSELF believes it's currently exporting, directly from its own runtime state — this is the server's own ground truth. `showmount -e <server>` run from a client queries the server remotely and shows what's actually being ADVERTISED and reachable from that client's network position — this can differ from the server's own `exportfs -v` view if there's a network-level issue (firewall blocking the NFS/rpcbind ports specifically from that client's location, per Day 6's firewalld coverage) preventing the client from even seeing the full export list the server believes it's offering. Comparing both lets you distinguish "the server genuinely isn't exporting this" (both views agree it's missing) from "the server IS exporting it, but something in the network path is preventing this specific client from seeing/reaching it" (server-side view shows it, client-side view doesn't) — two very different root causes requiring different fixes.

---

## Topic 2: AutoFS — Automounting & Map Files

### Quick Review
- AutoFS mounts a filesystem **on-demand**, the moment something accesses the mount point, and automatically unmounts it after a period of inactivity — rather than keeping every configured mount permanently active.
- `/etc/auto.master` is the top-level map, pointing to specific map files (e.g., `/etc/auto.misc`) for each managed mount point base.
- The `soft` vs `hard` NFS mount option (from Day 1/2) matters MORE here, since a hung automount attempt against an unreachable server can produce a stuck, unresponsive shell for anyone who happens to `cd` into that path.
- `automount -v` gives a verbose, foreground run — the correct tool for debugging a map file syntax issue rather than guessing from the service's normal background logs alone.

### Quick Learning

AutoFS solves a genuinely different problem than a static `/etc/fstab` NFS entry: rather than mounting every configured share at boot (consuming resources and creating a boot-time dependency on every referenced server being reachable), AutoFS mounts a share only the moment it's actually accessed, and unmounts it again after a period of disuse — reducing both resource usage and the blast radius of one unreachable server affecting unrelated boot/login processes. The tradeoff: the FIRST access to an AutoFS-managed path has to wait for the actual mount to happen in that moment, and if that mount attempt hangs (an unreachable server with a `hard` mount option), the process/shell that triggered it hangs right along with it.

**Static fstab mount vs. AutoFS on-demand mount:**
```
  Static /etc/fstab NFS entry               AutoFS-managed mount
  ─────────────────────────────              ────────────────────
  Mounted at BOOT TIME, unconditionally      NOT mounted until something
  (assuming "nofail" isn't masking a          actually ACCESSES the path
   failure) — consumes a mount slot and
   a live connection to the NFS server        cd /misc/data  ──▶ triggers
   for the ENTIRE uptime, whether or           the actual mount, ON DEMAND,
   not anyone ever actually uses it            at that exact moment

  If the NFS server is down at BOOT,          If the NFS server is down
  this can (per Day 1's fstab lesson)          WHEN SOMEONE FIRST ACCESSES
  block/delay boot entirely unless             the path, THAT access hangs
  "nofail" is set                              (if hard-mounted) — but boot
                                                 itself is never affected,
  Stays mounted FOREVER (until explicit         since nothing mounted anything
  unmount or reboot), consuming a live           at boot time at all
  connection even during long idle
  periods with zero actual use                After a period of inactivity,
                                                AutoFS automatically UNMOUNTS
                                                it again — no idle, unused
                                                connections lingering
```

### Implementation (Learn by Applying)

**Scenario:** Configure AutoFS to mount the NFS share from Topic 1 on-demand, confirm it correctly mounts on first access and unmounts after inactivity, and deliberately reproduce a map-file syntax error to practice diagnosing it with the correct tool.

```bash
dnf install -y autofs

cat >> /etc/auto.master << 'EOF'
/misc  /etc/auto.misc
EOF

cat >> /etc/auto.misc << 'EOF'
data  -rw,soft,intr  nfsserver:/data/shared
EOF

systemctl enable --now autofs
systemctl status autofs
```

Confirm on-demand behavior directly:
```bash
mount | grep "/misc/data"           # Should show NOTHING yet — not mounted until accessed
ls /misc/data                        # This access TRIGGERS the mount
mount | grep "/misc/data"           # NOW it appears

sleep 300                            # Wait past the default idle timeout
mount | grep "/misc/data"           # Should be gone again — automatically unmounted after inactivity
```

Deliberately break the map file syntax and diagnose it correctly:
```bash
echo "data2  BROKEN SYNTAX HERE nfsserver:/data/shared" >> /etc/auto.misc
systemctl restart autofs

ls /misc/data2 2>&1                  # Fails, but the error here is often unhelpfully generic

automount -v                          # Run in the foreground, verbose — THIS shows the actual map parsing error clearly
```
Fix the syntax and confirm:
```bash
sed -i '/BROKEN SYNTAX/d' /etc/auto.misc
systemctl restart autofs
```

### Interview Questions — with Answers

**1. Explain the fundamental operational difference between an AutoFS-managed mount and a static `/etc/fstab` NFS entry — not just "one is automatic," the actual resource and failure-mode implications.**

A static fstab entry is mounted unconditionally at boot and stays mounted for the system's entire uptime regardless of actual usage, meaning it consumes a live connection to the remote server continuously, and — without `nofail` — an unreachable server at boot time can block or delay the ENTIRE system's boot. An AutoFS-managed mount is only mounted the moment something actually accesses that path, and automatically unmounts after a period of inactivity — meaning boot time is never affected by whether that specific NFS server happens to be reachable (nothing tries to mount it at boot at all), and idle periods don't consume a lingering, unused connection. The tradeoff is that the FIRST access after a period of non-use has to wait for the actual on-demand mount to complete, introducing a small delay (or, in a failure scenario, a hang) at that moment that a permanently-mounted static entry wouldn't have.

**2. Why does the choice between `soft` and `hard` NFS mount options matter MORE for an AutoFS-managed mount than for many other NFS use cases?**

Because an AutoFS mount happens on-demand, TRIGGERED by whatever process/user happens to access that path at that specific moment — if the mount attempt hangs because the remote server is unreachable and the mount is `hard` (the default, retrying indefinitely), the specific process or interactive shell that triggered the access hangs right along with it, potentially without any indication to that user WHY their `cd` or `ls` command is simply not returning. This is a worse user experience than a static, already-established mount going stale (which at least was already mounted and working before something went wrong) — an AutoFS mount failing on FIRST access means the very act of trying to use the resource for the first time is what hangs, which is why `soft` (with an appropriate timeout, accepting the small data-integrity risk tradeoff discussed on Day 1/2) is frequently the more pragmatic choice specifically for AutoFS-managed, potentially-flaky remote mounts.

**3. A user reports that accessing an AutoFS-managed path takes a noticeable pause the first time in a while, but is instant on subsequent accesses. Is this a bug, and how would you explain it to them?**

This is expected, correct AutoFS behavior, not a bug — the first access after a period of inactivity is exactly when AutoFS performs the actual on-demand mount operation (network round-trip to the NFS server, mount negotiation, etc.), which introduces a real, if usually brief, delay; subsequent accesses within the "still mounted" window are instant because the mount is already established and just being used normally, like any other already-mounted filesystem. I'd explain this directly: AutoFS deliberately doesn't keep every configured mount active permanently specifically to avoid resource waste from unused connections, and the tradeoff for that efficiency is a brief pause on first-access-after-idle, which is the expected, by-design cost of the on-demand model rather than something indicating a problem.

**4. You've edited `/etc/auto.misc` to add a new map entry, but accessing the corresponding path still fails with a generic error. What's the correct diagnostic tool to use, and why is it better than just checking `systemctl status autofs`?**

`automount -v` run directly in the foreground gives verbose, real-time output showing exactly how AutoFS is parsing and attempting to use the map files, including specific syntax errors or parsing failures in the exact map entry causing trouble — this is significantly more diagnostic than `systemctl status autofs`, which shows general service health/state but typically doesn't surface the specific line-level parsing detail needed to spot a map file syntax mistake. `systemctl status` tells you the SERVICE is running; `automount -v` tells you WHAT the service is actually doing when it tries to process your specific, newly-added map entry, which is exactly the level of detail needed to catch a syntax typo or malformed entry.

**5. Explain the relationship between `/etc/auto.master` and files like `/etc/auto.misc` — why are these two separate, and what would break if you tried to put everything directly in `/etc/auto.master`?**

`/etc/auto.master` is the top-level map that tells AutoFS which base mount points exist and which SPECIFIC map file governs each one — it's a routing/dispatch table, not where individual mount definitions themselves live. `/etc/auto.misc` (or any other named map file referenced by `auto.master`) contains the actual, specific mount point definitions (name, options, remote source) for whatever base path `auto.master` pointed to it for. This separation allows different base mount points to be governed by entirely different, independently-manageable map files — useful for organizing many mount definitions logically (e.g., one map file per team or storage system) rather than one monolithic file, and allows different maps to potentially use different map TYPES (file-based, NIS-based, LDAP-based, program-based) per base path, a flexibility that would be lost if `auto.master` itself had to directly contain every individual mount definition rather than delegating to separate, purpose-specific map files.

---

## Topic 3: NFS Troubleshooting — Stale Mounts and Hangs

### Quick Review
- A "stale" NFS mount means the client's cached handle to server-side data no longer corresponds to anything valid (server rebooted, export removed, filesystem changed underneath) — commands against it return `Stale file handle`, not a hang.
- A HUNG NFS mount (distinct from stale) means the client is waiting on a `hard`-mounted, currently-unreachable server — commands don't error, they simply don't return.
- `umount -l` (lazy) detaches the mount point from the namespace immediately without needing the underlying connection to be healthy; `umount -f` (force) is for genuinely unresponsive hard mounts.
- `rpcinfo -p <server>` and `showmount -e <server>` confirm whether the SERVER side is actually the problem before assuming it's purely client-side.

### Quick Learning

"Stale" and "hung" are two DIFFERENT NFS failure modes that produce different symptoms and require different diagnostic approaches — conflating them is a common way to waste time troubleshooting the wrong thing. Stale gives you an immediate, clear ERROR (`Stale file handle`) — the client knows something's wrong and tells you so right away. Hung gives you SILENCE — the command just doesn't return, with no error at all, because the client is faithfully waiting (per a `hard` mount's design) for a server that isn't responding. Recognizing which one you're actually looking at, immediately, is the first real diagnostic step.

**Stale vs. hung — different symptoms, different causes, different fixes:**
```
  STALE FILE HANDLE                        HUNG (hard mount, server unreachable)
  ───────────────────                       ──────────────────────────────────
  Command returns IMMEDIATELY with           Command does NOT return at all —
  an explicit error:                         no error, no output, just... waits

  $ ls /mnt/nfsdata                          $ ls /mnt/nfsdata
  ls: cannot access '/mnt/nfsdata':          (hangs indefinitely, no prompt
  Stale file handle                           returns, Ctrl+C may not even
                                               work cleanly)

  CAUSE: server-side data/export             CAUSE: server is genuinely
  changed underneath an existing              unreachable (network down,
  client mount (server rebooted,              server crashed, firewall
  export reconfigured, underlying             blocking) AND the mount is
  filesystem remounted) — client's            `hard` (default), so the
  cached handle no longer maps to             client keeps RETRYING
  anything valid                              indefinitely rather than
                                               giving up
  FIX: umount and remount fresh               FIX: umount -f (force) or
  (client needs a NEW, valid handle)          umount -l (lazy) to detach
                                               without waiting for the
                                               server; investigate WHY the
                                               server is unreachable separately
```

### Implementation (Learn by Applying)

**Scenario:** Reproduce both failure modes distinctly — a genuine stale handle (by manipulating the server side underneath an active client mount) and a genuine hang (by making the server unreachable while hard-mounted) — and apply the correct recovery for each.

Reproduce a stale file handle:
```bash
# On the client, confirm the mount is currently healthy
ls /mnt/nfsdata

# On the SERVER, remove and recreate the export's underlying directory (simulating a real-world
# "someone recreated the filesystem underneath an active export" incident)
umount /data/shared 2>/dev/null    # if it was itself a separate mount
rm -rf /data/shared && mkdir /data/shared
exportfs -ra

# Back on the CLIENT, without unmounting first, try to use the now-stale handle
ls /mnt/nfsdata           # Expect: "Stale file handle" — immediate, explicit error
```
Fix:
```bash
umount /mnt/nfsdata
mount -t nfs nfsserver:/data/shared /mnt/nfsdata
ls /mnt/nfsdata            # Fresh handle, works normally now
```

Reproduce a genuine hang (lab-safe — simulate server unreachability via firewall, not actually destroying anything):
```bash
# On the SERVER, block NFS traffic entirely to simulate unreachability
firewall-cmd --add-rich-rule='rule family="ipv4" service name="nfs" reject' --timeout=120

# On the CLIENT, with a HARD-mounted share, attempt access
timeout 10 ls /mnt/nfsdata; echo "exit code: $?"    # Use timeout deliberately so the lab doesn't hang your terminal forever
```
Diagnose and recover correctly:
```bash
rpcinfo -p nfsserver 2>&1 | head -5     # Confirm SERVER-side unreachability specifically, not a client-only issue
showmount -e nfsserver 2>&1

umount -f /mnt/nfsdata 2>&1             # Force unmount, since the server genuinely isn't responding
# umount -l /mnt/nfsdata                # Lazy alternative — detaches immediately, cleans up once nothing references it
```

### Interview Questions — with Answers

**1. A user says an NFS-mounted directory is "not working." Before assuming anything, what's the very first thing you'd check to distinguish between a stale handle and a hang, and why does that distinction change your entire next steps?**

I'd have them (or check myself) run a simple, quick command against the mount (`ls` on the mount point) and observe: does it return IMMEDIATELY with an explicit error (`Stale file handle`), or does it simply not return at all (hanging)? This distinction fundamentally changes the diagnostic path — a stale handle means the client already knows something's wrong and just needs a fresh remount, a quick, low-risk fix; a hang means the underlying server is genuinely unreachable (or the network path to it is broken) and the mount is faithfully waiting per its `hard` configuration, requiring investigation into WHY the server is unreachable, and a `umount -f`/`umount -l` to escape the hang before you can even attempt further troubleshooting.

**2. Why doesn't a simple `umount` (without `-f` or `-l`) work when a hard-mounted NFS share's server has become unreachable — what's actually happening under the hood?**

A plain `umount` attempts a clean, cooperative unmount, which for NFS can involve communicating with the server as part of a graceful teardown, or at minimum waiting for any in-flight operations against that mount to resolve first — but if the server is genuinely unreachable and the mount is `hard` (meaning the kernel is configured to retry indefinitely rather than give up), there may be operations perpetually "in flight" from the kernel's perspective, waiting for a server response that will never come, which can cause a plain `umount` to itself hang or fail rather than completing. `umount -f` forces the unmount without waiting for that cooperative teardown to succeed, and `umount -l` (lazy) detaches the mount point from the filesystem namespace IMMEDIATELY, cleaning up the actual underlying connection asynchronously once nothing is still referencing it — both bypass the need for a healthy, responsive server to complete the unmount, which is exactly what's needed when the server itself is the problem.

**3. Explain the actual difference between `umount -f` and `umount -l`, and describe a scenario where you'd specifically prefer one over the other.**

`umount -f` (force) attempts to forcibly complete the unmount operation immediately, which can be more disruptive to any processes that currently have open file handles or are mid-operation against that mount — those operations may fail abruptly. `umount -l` (lazy) detaches the mount point from the filesystem namespace right away, so no NEW access to that path is possible, but the actual underlying mount/connection isn't fully torn down until whatever currently has it open finishes or is otherwise cleaned up — a gentler approach that lets in-flight operations resolve (or fail) more naturally rather than being forcibly interrupted. I'd prefer `-l` when I want to quickly free up the mount point (e.g., to remount fresh) without necessarily knowing or caring about exactly what's currently using it, and reserve `-f` for situations where I need genuine certainty that the mount is gone immediately, even at the cost of forcibly disrupting whatever's using it.

**4. A stale file handle error appears on a client, but the NFS server itself was never rebooted, and nobody touched the export configuration. What ELSE could cause this?**

Stale handles aren't exclusively caused by a server reboot or export reconfiguration — they can also occur if the underlying filesystem or specific files/directories on the server were modified in a way that invalidates the client's cached reference, such as a directory being deleted and recreated with the same name (a NEW inode, even though the path looks identical to an observer), the underlying LOCAL filesystem on the server being remounted or having an fsck run against it, or in more complex environments, the server-side storage itself (a SAN LUN, in an environment like yours) being reconfigured or presented differently underneath the export without the NFS service being explicitly restarted — from the client's perspective, its cached file handle simply no longer corresponds to valid data, regardless of the SPECIFIC underlying cause on the server side, which is why "stale" is deliberately a generic symptom covering several possible underlying triggers.

**5. How would you distinguish "the NFS server itself is completely down" from "the server is up, but this SPECIFIC export/share has an issue" when troubleshooting from a client?**

`rpcinfo -p <server>` confirms whether the server's RPC/NFS services are running and reachable AT ALL — if this fails entirely, the server (or at minimum, its NFS-related services) is genuinely down or unreachable at the network level, a broader problem than any specific export. If `rpcinfo` succeeds (confirming the server and its NFS services are up and reachable), but `showmount -e <server>` doesn't show the specific export you're trying to reach, or shows it with unexpected options, that narrows the problem down to something specific to THAT export's configuration rather than the server as a whole — perhaps it was removed from `/etc/exports`, never had `exportfs -ra` run after being added, or has access restrictions (client subnet mismatch) preventing your specific client from seeing it. This layered check — server reachability first, then specific export visibility — mirrors the same "isolate which layer the problem is actually at" discipline from Day 2's connectivity troubleshooting escalation order.

---

## Topic 4: Chrony/NTP & timedatectl — Time Sync and Downstream Impact

### Quick Review
- `chronyc sources -v` shows configured time sources with stratum, reachability, and offset; `chronyc tracking` shows the CURRENT sync accuracy.
- Clock drift breaks Kerberos/AD authentication (tight time-window tolerances), produces misleading cross-host log correlation, can cause cron jobs to run at unexpected times relative to wall-clock expectations, and can break TLS certificate validation.
- `timedatectl` manages timezone and whether NTP sync is enabled at all; `chronyc` manages the actual sync daemon's behavior and diagnostics.
- Virtualized hosts have their OWN specific clock-drift risk (hypervisor scheduling, VM pause/resume) beyond typical physical-hardware clock drift causes.

### Quick Learning

Time sync is one of those topics that seems almost too basic to ask about at a senior level, which is exactly why interviewers DO ask about it — the real test isn't "do you know what NTP is," it's "can you connect clock drift to its actual, sometimes surprising downstream failures" (auth, logs, TLS, scheduling), because a candidate who's only ever thought about NTP as "keeps the clock right" hasn't actually had to debug the kind of incident where drift was the ROOT cause of something that looked completely unrelated on the surface.

**One root cause, several unrelated-looking symptoms:**
```
                    Server clock drifts significantly
                              │
        ┌─────────────────────┼─────────────────────┬──────────────────┐
        ▼                     ▼                     ▼                  ▼
  Kerberos/AD auth      Cross-host log            TLS cert            Cron jobs run
  starts failing          correlation               validation          at "wrong"
  (tickets have            becomes unreliable        starts failing      times relative
  tight time-window        (event ordering            ("not yet valid"    to OTHER
  tolerances, often         across hosts no            / "expired" even    systems'
  ~5 min by default)        longer makes sense          for genuinely       expectations,
                            when their clocks             valid certs)      even though the
                            disagree)                                      job itself ran
                                                                            "on schedule"
                                                                            per ITS OWN
                                                                            drifted clock

  Someone investigating ANY ONE of these symptoms in isolation
  might spend significant time on auth-specific, logging-specific,
  or TLS-specific troubleshooting before ever checking `chronyc
  tracking` — which is exactly why "what breaks when NTP is down"
  is worth having a crisp, complete mental list for.
```

### Implementation (Learn by Applying)

**Scenario:** Diagnose a reported "intermittent Kerberos authentication failures" ticket that's actually caused by clock drift — practice recognizing the connection rather than troubleshooting the auth stack directly first.

```bash
timedatectl status
chronyc tracking
chronyc sources -v
```
Focus on `chronyc tracking`'s key fields: **System time offset** (how far off the local clock currently is from the reference), and **RMS offset** (a longer-term measure of drift/jitter) — a meaningfully large or actively growing offset here is the smoking gun for exactly this class of symptom.

Simulate drift deliberately (lab-safe, deliberately breaking time sync to observe the effect):
```bash
systemctl stop chronyd
date -s "+10 minutes"                # Artificially skew the clock forward
date

# Attempt a Kerberos-authenticated operation (conceptually — kinit against a KDC with a tight clock-skew tolerance)
kinit testuser 2>&1 | grep -i "clock skew\|time"
```

Diagnose and fix correctly:
```bash
chronyc tracking                     # Confirm chronyd isn't even running right now — the actual root cause
systemctl start chronyd
chronyc sources -v
chronyc makestep                     # Force an immediate, large correction rather than chronyd's normal gradual slew,
                                       # appropriate right after a large, deliberate/accidental time jump like this lab
chronyc tracking                      # Confirm offset has returned to near-zero
```

### Interview Questions — with Answers

**1. A team reports intermittent Kerberos authentication failures, and initial investigation focuses on the KDC and auth configuration with no findings. What would make you suspect clock drift specifically, and how would you confirm it quickly?**

Kerberos tickets are cryptographically bound to a specific time window (commonly a default ~5 minute tolerance), and authentication failures that are "intermittent" in a way that doesn't correlate cleanly with any auth-configuration change or KDC issue is a classic signature of clock drift — since whether an operation succeeds or fails can depend on the EXACT moment the clocks happen to be far enough apart, rather than a consistent, always-broken configuration issue. I'd quickly confirm by running `chronyc tracking` on both the client and the KDC (or whatever systems are involved in the failing auth flow), checking the reported system time offset — a meaningfully large offset on either side, especially if it's actively growing rather than stable, strongly confirms drift as the actual root cause rather than continuing to investigate the auth configuration itself, which was likely never the actual problem.

**2. Explain why clock drift specifically undermines cross-host log correlation, beyond just "the timestamps are wrong." Give a concrete example of how this actually misleads an investigation.**

When investigating an incident spanning multiple hosts, the ORDER in which events occurred across those hosts is often critical to understanding cause and effect — did host A's error happen BEFORE or AFTER host B's related event? If the hosts' clocks have drifted apart from each other, their respective log timestamps no longer accurately reflect the TRUE, real-world chronological order of events, even though each individual host's own internal log sequence is still correctly ordered relative to ITSELF. Concrete example: investigating a cascading failure where Host A's timestamp shows an error at 10:00:05 and Host B's related failure shows 10:00:03 — appearing to suggest Host B failed FIRST and caused Host A's issue — but if Host B's clock is actually running 10 seconds fast relative to true time (and Host A's clock is accurate), the TRUE order was actually the reverse, and an investigator trusting the raw timestamps without accounting for drift would draw exactly the wrong conclusion about cause and effect.

**3. Why can clock drift cause TLS certificate validation failures even for certificates that are genuinely, currently valid according to their actual issued validity window?**

TLS certificate validation checks the certificate's "not before" and "not after" validity dates against the LOCAL system's OWN current clock, not against some independently-verified true time — if the local system's clock is drifted significantly (particularly backward, or forward past a certificate's actual expiration), the validation check compares the certificate's real, correct validity window against an INCORRECT local reference point, producing a "not yet valid" or "expired" error even though the certificate is, by its actual issued dates, genuinely fine. This is a subtle failure mode because the certificate itself, the CA, and the actual calendar date are all correct — the ENTIRE problem is the local system's own clock being wrong, which is easy to overlook when troubleshooting what looks like a certificate problem, since certificate errors intuitively suggest investigating the certificate itself first, not the local clock.

**4. What's the practical difference between `timedatectl set-ntp true` and directly managing `chronyd` via `systemctl`/`chronyc` — when would you use each?**

`timedatectl` is a higher-level, broader system-time management tool that controls whether NTP synchronization is enabled AT ALL (as a general system setting) alongside other time-related settings like timezone — `set-ntp true/false` is essentially a top-level on/off switch for time synchronization as a concept. `chronyc` and direct `systemctl` management of `chronyd` operate at a more granular level — configuring/inspecting the SPECIFIC sync daemon's behavior, sources, current tracking accuracy, and forcing specific corrective actions (like `chronyc makestep`). In practice, I'd use `timedatectl` for the simple, top-level "is NTP sync supposed to be on for this system at all" question and general timezone management, and drop into `chronyc`/`systemctl chronyd` specifically when I need to actually diagnose sync ACCURACY, source health, or force a specific corrective behavior — `timedatectl status` even surfaces a summary of chrony's sync state for convenience, but the deeper diagnostic detail lives in `chronyc` specifically.

**5. Beyond typical physical-hardware clock drift causes, what's a clock-drift risk specific to VIRTUALIZED hosts that a SysAdmin with a Power Systems/virtualization background should be particularly alert to?**

Virtualized guests are subject to drift risks that don't apply the same way to physical hardware, specifically around HYPERVISOR SCHEDULING — if a VM is paused, migrated (live migration), or simply doesn't receive consistent CPU scheduling time from the hypervisor for a period (e.g., host oversubscription/contention), the guest's own sense of elapsed time can diverge meaningfully from true wall-clock time in ways a physical machine's hardware clock wouldn't experience under normal operation. This is exactly why virtualization platforms typically provide (and RHEL guests typically use) a paravirtualized clock source when running as a VM, and why NTP/chrony sync is arguably even MORE important, not less, in virtualized environments — a guest's local clock hardware is fundamentally less trustworthy as a stable time reference than physical hardware, making active, well-tuned synchronization the primary defense against drift rather than a nice-to-have.

---

## Topic 5 (Bonus — Advanced): NFSv4 Pseudo-Filesystem, ID Mapping, and Kerberized NFS

### Quick Review
- NFSv4 exposes a single, unified **pseudo-filesystem root** — clients see one coherent namespace rooted at the server's export root, rather than mounting each export as an independent filesystem the way NFSv3 requires.
- `rpc.idmapd` (`nfsidmapd`) translates between NUMERIC UIDs/GIDs (used on the wire in NFSv2/v3) and NAME-based identities (`user@domain`) used by NFSv4 — a mismatch here produces files showing as owned by `nobody` even when permissions/exports are otherwise correct.
- `sec=sys` (the traditional, weaker trust-the-client-supplied-UID model) versus `sec=krb5`/`krb5i`/`krb5p` (genuine Kerberos-authenticated, and optionally integrity/privacy-protected, NFS) represent fundamentally different trust models for who the NFS server believes is making a request.
- This topic ties directly back to Day 5's NFSv4 ACL bonus content — the richer NFSv4 ACL model and this pseudo-filesystem/security model are part of the same NFSv4 protocol generation's broader redesign versus NFSv3.

### Quick Learning

NFSv4 wasn't just an incremental version bump — it's a genuinely redesigned protocol, and the pseudo-filesystem model plus its identity/security options are where that redesign shows up most operationally. The single unified namespace simplifies client mounting (one mount point can expose everything the server chooses to present beneath it, rather than needing separate mounts per export), but it introduces a NEW failure mode: NFSv4's move from purely numeric UID/GID identification (NFSv3's model, where the client simply asserts "I am UID 1000" and the server trusts it) to NAME-based identification requires a working ID-mapping service on both client and server, and a broken or misconfigured `idmapd` is a genuinely common, specific NFSv4 troubleshooting scenario that doesn't exist in quite the same form under NFSv3.

**NFSv3's per-export mounting vs. NFSv4's single pseudo-filesystem root:**
```
  NFSv3 — each export mounted independently:
     mount server:/export/data1   /mnt/data1
     mount server:/export/data2   /mnt/data2
     mount server:/export/logs    /mnt/logs
     (three separate mount operations, three separate
      NFSv3 "filesystems" from the client's perspective)

  NFSv4 — one pseudo-filesystem root, everything beneath it:
     mount server:/            /mnt/allexports
     ls /mnt/allexports
       data1/  data2/  logs/
     (ONE mount operation; the server's pseudo-filesystem
      presents a unified view of everything it chooses to
      export beneath its root — individual exports appear
      as subdirectories of this single coherent namespace)
```

**Where ID mapping fits, and what breaks when it's misconfigured:**
```
  NFSv3 (numeric, client-asserted):        NFSv4 (name-based, requires idmapd):
  ─────────────────────────────────         ──────────────────────────────────
  Client says: "this file is UID 1000"      Client says: "this file is owned
  Server TRUSTS this numeric claim           by alice@example.com"
  directly — no translation needed,
  but also no independent verification      Server's rpc.idmapd translates
  beyond "does this UID exist locally"       alice@example.com ⟷ local UID 1000
                                              (and the CLIENT'S idmapd does the
                                              same translation in reverse)

  If idmapd is misconfigured or the domain configured in
  /etc/idmapd.conf doesn't match between client and server:
     ⇒ files show as owned by "nobody" / a numeric ID that
       doesn't map to any recognizable name, EVEN THOUGH the
       underlying permissions/exports are otherwise entirely
       correct — a classic, specific NFSv4 troubleshooting trap
```

### Implementation (Learn by Applying)

**Scenario:** Configure an NFSv4 export and mount using the pseudo-filesystem model, then deliberately misconfigure `idmapd`'s domain setting to reproduce and diagnose the classic "everything shows as nobody" symptom — and finally, understand (via configuration inspection, since a full Kerberos KDC lab is out of scope here) the security-model difference `sec=krb5` represents.

```bash
# Server-side: confirm NFSv4 pseudo-filesystem export behavior
cat /etc/exports
# /data/shared  10.0.0.0/24(rw,sync,root_squash)
exportfs -v

# Client-side: mount using NFSv4 explicitly, note there's no need to specify the FULL server-side path structure separately per export
mount -t nfs4 nfsserver:/ /mnt/allexports
ls /mnt/allexports                     # Should show "shared" (or whatever subdirectory structure the server's exports create)
```

Reproduce the classic idmapd domain-mismatch symptom:
```bash
cat /etc/idmapd.conf | grep -i domain
# On the SERVER:
sed -i 's/^Domain = .*/Domain = example.com/' /etc/idmapd.conf
systemctl restart nfs-idmapd 2>/dev/null || systemctl restart rpc-idmapd

# On the CLIENT, deliberately set a DIFFERENT, mismatched domain
sed -i 's/^Domain = .*/Domain = wrongdomain.com/' /etc/idmapd.conf
systemctl restart nfs-idmapd 2>/dev/null || systemctl restart rpc-idmapd

# Create a file on the server side, owned by a real, mapped user, then view it from the client
touch /data/shared/domain_test_file.txt
chown alice:alice /data/shared/domain_test_file.txt

ls -l /mnt/allexports/shared/domain_test_file.txt    # On the client — expect "nobody" or numeric-only ownership,
                                                        # DESPITE the file genuinely being owned by alice on the server
```
Fix by matching the domain configuration on both sides:
```bash
sed -i 's/^Domain = .*/Domain = example.com/' /etc/idmapd.conf   # on the client, matching the server now
systemctl restart nfs-idmapd 2>/dev/null || systemctl restart rpc-idmapd
ls -l /mnt/allexports/shared/domain_test_file.txt    # Should now correctly show "alice"
```

Understand the `sec=` options directly (conceptual inspection, given a full Kerberos KDC setup is beyond this lab's scope):
```bash
mount -t nfs4 -o sec=sys nfsserver:/ /mnt/allexports_sys      # Traditional, default — trusts client-asserted identity
# mount -t nfs4 -o sec=krb5  nfsserver:/ /mnt/allexports_krb5   # Requires a working Kerberos realm — genuine authentication
# mount -t nfs4 -o sec=krb5i nfsserver:/ /mnt/allexports_krb5i  # krb5 + per-packet INTEGRITY protection
# mount -t nfs4 -o sec=krb5p nfsserver:/ /mnt/allexports_krb5p  # krb5 + per-packet PRIVACY (encryption)
```

### Interview Questions — with Answers

**1. Explain the practical difference between NFSv3's per-export mounting model and NFSv4's pseudo-filesystem model — what changes operationally for a client mounting multiple shares from the same server?**

Under NFSv3, each export the server offers has to be mounted independently and explicitly by the client — accessing three separate exports means three separate `mount` operations, each treated as its own distinct filesystem from the client's perspective. Under NFSv4, the server presents a single, unified pseudo-filesystem rooted at its export root, and a client can mount just that ROOT once, seeing every export the server chooses to present beneath it as a coherent subdirectory structure within that single mount — operationally simplifying client-side configuration (fewer mount operations/fstab entries needed) and presenting a more unified, filesystem-like view of everything the server offers, rather than a collection of independently-mounted, unrelated-looking filesystems.

**2. A file on an NFSv4 mount shows as owned by "nobody" on the client, despite the server clearly showing it owned by a legitimate, real user, and the export's permissions/options all appear correct. What's the most likely cause, and how do you confirm it?**

The most likely cause is an `rpc.idmapd` domain mismatch between the client and server — NFSv4 uses name-based identity (`user@domain`) rather than NFSv3's purely numeric UID model, and this requires the ID-mapping service's configured domain to match between client and server for the translation to succeed; a mismatch causes the mapping to fail, and the server (or client, depending on where the mapping breaks down) falls back to presenting the file as owned by the unmapped/unknown identity, `nobody`. I'd confirm by checking `/etc/idmapd.conf`'s `Domain =` setting on BOTH the client and server and comparing them directly for an exact match — a mismatch here, exactly as reproduced in the lab, is the classic, specific signature of this particular NFSv4 issue.

**3. What's the fundamental trust-model difference between `sec=sys` and `sec=krb5` for an NFSv4 mount — what is the server actually trusting in each case?**

With `sec=sys` (the traditional default), the server trusts whatever UID/GID the CLIENT itself asserts as part of its request — there's no cryptographic verification that the client is genuinely who/what it claims; a compromised or maliciously-configured client could simply assert a different UID than its actual user and the server would have no independent way to detect this. With `sec=krb5`, the client's identity is genuinely, cryptographically AUTHENTICATED via Kerberos as part of establishing the NFS session — the server isn't trusting a bare, unverified claim, it's relying on Kerberos's actual authentication guarantees, making `sec=krb5` (and its stronger variants `krb5i`/`krb5p`, adding per-packet integrity/privacy protection respectively) meaningfully more secure against a compromised or spoofing client than `sec=sys`'s inherent trust-the-client model.

**4. Why might an organization choose `sec=krb5i` specifically over plain `sec=krb5`, given that both provide genuine Kerberos authentication?**

Plain `sec=krb5` authenticates the CLIENT's identity at session establishment, but doesn't provide any ongoing protection for the actual data packets exchanged during the NFS session afterward — a network-level attacker who can intercept traffic could still potentially tamper with packets in transit even though the original authentication was genuine. `sec=krb5i` adds per-packet INTEGRITY protection on top of the same Kerberos authentication, cryptographically ensuring that packets haven't been tampered with in transit, without going as far as full encryption (`krb5p`) — this is a reasonable middle-ground choice for organizations that want protection against active tampering/man-in-the-middle attacks on NFS traffic, but don't need (or want to accept the performance cost of) full payload encryption, which `krb5p` would add on top.

**5. This topic connects directly back to Day 5's NFSv4 ACL bonus content. Explain the conceptual link — why do NFSv4's richer ACL model, pseudo-filesystem, and Kerberized security options all belong to "the same" protocol redesign rather than being unrelated, independently-added features?**

NFSv4 represented a genuinely comprehensive protocol redesign compared to NFSv3's more incremental, RPC-based approach — the richer ACL model (with granular ALLOW/DENY entries and explicit inheritance flags, as covered on Day 5), the unified pseudo-filesystem namespace, and integrated strong security (Kerberos-based authentication with optional integrity/privacy) were all designed together as part of the SAME broader effort to modernize NFS into a more capable, more secure, more coherent protocol for a single client-server session — rather than being separately bolted-on features added at different times for unrelated reasons. Recognizing this connection matters practically: someone deploying NFSv4 seriously is likely to encounter ALL of these related considerations together (richer ACLs to manage, a unified namespace to design exports around, and a security model decision to make between `sec=sys` and Kerberized options) as part of one coherent NFSv4 adoption decision, not as a series of unrelated, independently-optional features to consider in isolation.

---

**End of Day 8.** You should now be able to configure NFS exports with production-appropriate security defaults (understanding exactly why `root_squash` matters), build AutoFS-managed on-demand mounts and correctly reason about their resource/failure-mode tradeoffs versus static fstab entries, immediately distinguish a stale handle from a genuine hang and apply the correct recovery for each, connect clock drift to its full range of downstream symptoms (auth, logs, TLS, scheduling) rather than troubleshooting each in isolation, and — as the advanced capstone — reason about NFSv4's pseudo-filesystem model, diagnose the classic idmapd domain-mismatch symptom, and explain the genuine trust-model difference Kerberized NFS provides over the traditional `sec=sys` default.

Proceed to **Day 9 — Recovery & Automation Bonus** next.
