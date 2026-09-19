# Day 9 — Recovery & Automation (Full Study Edition, with Answers)

**Estimated time: 5–6 hours**

Today is the "commonly asked, easy to under-prepare for" day — password/root recovery genuinely does get asked at product companies despite feeling like RHCSA trivia, because it's a real test of whether you understand the boot process well enough to intervene in it; cron/at, logrotate, shell scripting, and sysctl are the unglamorous automation and tuning fundamentals that come up constantly in "tell me about a time you automated something" and "how would you tune this server" style questions. Basic-to-advanced across all five topics, as requested.

---

## Topic 1: Password Reset & Root Recovery — Single-User Mode, Rescue Mode, GRUB

### Quick Review
- The core technique: interrupt GRUB at boot, edit the kernel command line to boot directly into a shell (`rd.break` or appending `init=/bin/bash` / `systemd.unit=rescue.target` depending on the exact scenario), bypassing normal password authentication entirely.
- `rd.break` drops you into the **initramfs** environment, BEFORE the real root filesystem is even mounted — the most powerful/earliest intervention point, needed when normal boot can't even get far enough to reach a shell any other way.
- After changing the password from this pre-boot environment, an SELinux relabel is often required (per Day 7's disabled-mode lesson) because files touched from the initramfs environment may not get correctly labeled the way a normal running system would.
- This entire technique existing and working is exactly why physical/console (BIOS/GRUB) access security matters — anyone with unrestricted console access can reset root's password without ever knowing the original one.

### Quick Learning

The RHCSA-level version of this is "memorize the steps." The product-company-interview version is understanding WHY each step is necessary — specifically, why you can't just boot normally and reset the password the usual way (because you're locked out and don't know it), why `rd.break` intervenes so early (before the root filesystem is even mounted, meaning before ANY normal userspace tooling, including `passwd`, is available), and why the relabel step exists (SELinux context correctness depends on the labeling process having actually run, and doing filesystem writes from the minimal initramfs environment can bypass that).

**Where in the boot sequence each recovery technique actually intervenes:**
```
  Normal boot sequence:
  GRUB ──▶ kernel + initramfs loads ──▶ initramfs finds and
  mounts REAL root filesystem ──▶ switches to it (switch_root)
  ──▶ systemd starts ──▶ normal targets (sysinit, basic,
  multi-user, etc.) ──▶ login prompt

        ▲                    ▲                          ▲
        │                    │                          │
   rd.break intervenes   systemd.unit=rescue.target   emergency mode
   HERE — before the     intervenes HERE — after       (Day 3) can be
   real root FS is       the real root is mounted      reached via
   even mounted; you're  and switched to, but before    various boot
   in the INITRAMFS      normal multi-user services      FAILURES, not
   shell, root FS is     start — most of the system       something you
   just sitting there    is usable, closer to a           deliberately
   UNMOUNTED, waiting     genuinely normal environment      invoke for
                                                             password reset

  rd.break is the technique for "I need to fix something
  about the root filesystem ITSELF before it's even usable" —
  exactly the situation when you don't know root's password
  and need to reset it before ANYTHING else can happen.
```

### Implementation (Learn by Applying)

**Scenario:** You've genuinely lost the root password on a RHEL 9 VM (a real, common scenario — inherited server, forgotten credentials, ex-employee situation) and need to recover access without reinstalling.

At the GRUB menu (interrupt the countdown by pressing any key, then `e` to edit the selected entry):
```
# Find the line starting with "linux" (the kernel command line) and append:
rd.break
# Then press Ctrl+X (or F10) to boot with this modification
```

At the resulting initramfs shell:
```bash
# The real root filesystem is mounted read-only at /sysroot at this point - remount it read-write
mount -o remount,rw /sysroot

# chroot into it to operate as if it were the real, running root filesystem
chroot /sysroot

passwd root
# Enter the new password when prompted

# CRITICAL: touch the autorelabel flag so SELinux contexts get corrected on next boot,
# since files touched in this minimal environment may not be labeled correctly
touch /.autorelabel

exit                # Leave the chroot
exit                # Leave the initramfs shell, resuming normal boot with the new password set
```

After reboot (the autorelabel will make this boot take noticeably longer than usual — expected):
```bash
getenforce             # Confirm the system came back up in its expected mode after the relabel
passwd -S root          # Confirm the password was actually changed successfully
```

Understand the alternative, less-early intervention point for comparison:
```
# At the SAME GRUB edit screen, an alternative approach for RHEL 8/9:
# append instead:  rd.break enforcing=0
# (boots to the SAME initramfs break point, but pre-emptively sets permissive mode
#  for the FIRST boot after the fix, avoiding a full autorelabel in some scenarios —
#  understand this as an alternative, not necessarily always the better choice,
#  since it leaves the system briefly permissive rather than doing a clean relabel)
```

### Interview Questions — with Answers

**1. Walk me through the ENTIRE root password recovery process on RHEL 9, and explain WHY each step is necessary, not just what to type.**

At the GRUB menu, I'd interrupt the boot countdown and edit the kernel command line to append `rd.break`, which causes the boot process to intentionally halt at the initramfs stage, BEFORE the real root filesystem has been mounted and switched to — necessary because at this point, no normal userspace authentication or tooling exists yet, so there's no "locked out" state to work around, since normal boot hasn't progressed far enough to enforce any login at all. From the resulting initramfs shell, the real root filesystem is present but mounted read-only at `/sysroot` — I remount it read-write (`mount -o remount,rw /sysroot`) since password changes require write access, then `chroot /sysroot` to operate within that filesystem as if it were the actual running system, allowing `passwd root` to work normally against the real `/etc/shadow`. Finally, `touch /.autorelabel` before exiting is necessary because SELinux relies on consistent labeling maintained during normal operation, and changes made from this minimal, pre-normal-boot environment aren't guaranteed to have been labeled correctly — the autorelabel flag ensures a full relabel happens on the next normal boot, restoring correct SELinux contexts throughout the filesystem.

**2. Why does `rd.break` intervene BEFORE the root filesystem is mounted, rather than using a technique that boots further into the system first?**

Because the specific problem being solved — not knowing root's password — genuinely requires access to modify `/etc/shadow`, and if boot were allowed to progress further into a NORMAL environment, it would enforce the very authentication you're trying to bypass (you'd hit a login prompt requiring the password you don't have). `rd.break`'s early intervention point exists specifically because it sidesteps normal authentication entirely by never actually reaching the point where authentication would be enforced — you're operating in a pre-boot, effectively "nothing has started yet, I'm just a shell with access to the raw filesystem" environment, which is the earliest point where you can meaningfully make changes but also the point specifically designed to have no login barrier of its own to bypass.

**3. What actually happens if you forget the `touch /.autorelabel` step, and how would that failure manifest itself after the reboot?**

Without the relabel flag, the system boots normally after the password reset, but any files/metadata touched during the recovery process (potentially including `/etc/shadow` itself, now that its content has changed) may not have correct, consistent SELinux context labels — since the initramfs environment isn't running the normal SELinux-aware processes that maintain correct labeling during regular operation. This can manifest as SELinux denials on subsequent normal boots — services failing to start, or specific operations against the affected files being blocked — that look like an unrelated, confusing new problem, when the actual root cause traces back to files touched during the password-recovery process never having been properly relabeled afterward. It's exactly the kind of "fixed one problem, silently created a different one" trap that comes from skipping a step whose PURPOSE (SELinux label consistency) isn't obviously connected to the task at hand (resetting a password) unless you understand why it's there.

**4. What's the security implication of this entire recovery procedure existing and working reliably — what does that mean for how seriously physical/console access security should be taken?**

This procedure fundamentally means that anyone with unrestricted physical or console-level access to a server (the ability to interrupt GRUB and edit boot parameters) can reset root's password WITHOUT ever knowing the original password — meaning physical/console security is, in an important sense, the ACTUAL security boundary for a server, not the root password itself, which can always be bypassed given this level of access. This is exactly why serious security postures include GRUB password protection (requiring a separate password just to EDIT boot parameters, closing this specific bypass), physical datacenter access controls, and BIOS/console-level access restrictions (IPMI/iDRAC credentials, virtualization platform access controls) as genuinely critical security layers — treating "someone would need physical/console access to exploit this" as a low-risk caveat rather than a real, actively-defended security boundary is a common and serious misjudgment.

**5. How would you PREVENT this exact recovery technique from being usable by an unauthorized person who gains physical/console access to a production server, and what's the tradeoff of doing so?**

I'd configure a GRUB password (`grub2-setpassword` or the equivalent mechanism for the specific RHEL version), which requires a separate credential just to EDIT boot parameters at the GRUB menu — without knowing THIS password, an attacker with console access can still boot the system normally, but cannot modify the kernel command line to append `rd.break` or similar recovery/bypass techniques, closing off this specific vector. The tradeoff: this ALSO means legitimate administrators lose easy access to this exact recovery procedure if they themselves get locked out and don't have the GRUB password documented/available — creating a genuine operational risk of a fully-locked-out system if the GRUB password itself is lost, which is why organizations doing this typically need a carefully managed, separately-secured process for storing and accessing GRUB passwords (or an alternative out-of-band recovery process) alongside implementing this protection, rather than treating it as a purely one-sided security improvement with no operational cost.

---

## Topic 2: Cron & At Scheduling — Basic Syntax to Environment Pitfalls

### Quick Review
- Cron syntax: `minute hour day-of-month month day-of-week command` — five fields, each accepting a number, range, list, step (`*/N`), or wildcard.
- `crontab -e` edits the CURRENT user's crontab; `crontab -e -u username` (as root) edits another user's; `/etc/cron.d/` holds system-wide, non-per-user cron files with an ADDITIONAL user field.
- Cron jobs run in a minimal environment — NOT sourcing `~/.bashrc`, and typically with a much smaller `$PATH` than an interactive shell — the single most common source of "works when I run it manually, fails from cron."
- `at` schedules a ONE-TIME future job (versus cron's recurring schedule) — genuinely useful for deferred one-off actions, including the "scheduled rollback" safety technique from Day 2.

### Quick Learning

Basic cron syntax is memorization; the advanced, actually-interview-relevant knowledge is understanding WHY a script that "works fine when I run it manually" mysteriously fails or behaves differently when the exact same script runs from cron — and the answer is almost always environment differences, not the script's logic itself. Cron's execution environment deliberately doesn't replicate an interactive login shell's environment (no sourced `.bashrc`, a minimal `$PATH` often limited to just `/usr/bin:/bin`, no inherited environment variables an interactive session might have set) — a script relying on any of that implicitly is fragile in a way that only surfaces specifically when run via cron.

**Why "works manually, fails from cron" is almost always an environment problem:**
```
  Interactive shell (what you test with)      Cron's execution environment
  ──────────────────────────────────────       ─────────────────────────────
  Sources ~/.bashrc, ~/.bash_profile           Does NOT source any of these
  (any PATH additions, aliases, exported
   variables YOU'VE customized are active)     $PATH is often just a minimal
                                                 default (e.g. /usr/bin:/bin)
  $PATH includes whatever YOUR shell
  config added (e.g. /usr/local/bin,           A command that works because
  a custom tools directory, etc.)               it's in YOUR PATH via .bashrc
                                                  customization silently FAILS
  Any environment variables an app relies       from cron with "command not
  on (API keys, custom config paths) that        found" — even though the
  you've exported in your shell session          SAME script, SAME user, works
  are present                                    fine when run interactively

  FIX: never rely on inherited environment — either use FULL, EXPLICIT
  paths for every command inside scripts, or explicitly source needed
  environment/set PATH at the TOP of the script itself, so it's
  self-contained regardless of what environment invoked it (the same
  principle Day 4's umask topic established for exactly this reason)
```

### Implementation (Learn by Applying)

**Scenario:** Reproduce the classic "works manually, fails from cron" bug directly, diagnose it correctly, and fix it with the robust, self-contained approach — then schedule a genuine one-time deferred task with `at`, including the safety-net pattern from Day 2.

```bash
# Add a custom tool to a directory NOT in cron's minimal default PATH, but IS in your interactive shell's PATH
mkdir -p /opt/customtools
cat > /opt/customtools/mytool.sh << 'EOF'
#!/bin/bash
echo "mytool ran successfully"
EOF
chmod +x /opt/customtools/mytool.sh
echo 'export PATH=$PATH:/opt/customtools' >> ~/.bashrc
source ~/.bashrc

# Write a script that relies on this tool being findable via PATH, without an explicit full path
cat > /usr/local/bin/cron_test_script.sh << 'EOF'
#!/bin/bash
mytool.sh >> /tmp/cron_test_output.log 2>&1
EOF
chmod +x /usr/local/bin/cron_test_script.sh

# Confirm it works fine when run manually, interactively
/usr/local/bin/cron_test_script.sh
cat /tmp/cron_test_output.log
```

Now run the identical script via cron and observe the failure:
```bash
echo "* * * * * /usr/local/bin/cron_test_script.sh" | crontab -
sleep 65
cat /tmp/cron_test_output.log     # Likely shows "command not found" for mytool.sh — cron's PATH never had /opt/customtools
crontab -r                          # Clear the test cron entry
```

Fix it the robust way — never depend on inherited PATH/environment inside a script meant to run unattended:
```bash
cat > /usr/local/bin/cron_test_script.sh << 'EOF'
#!/bin/bash
/opt/customtools/mytool.sh >> /tmp/cron_test_output.log 2>&1    # FULL, explicit path — works regardless of caller's PATH
EOF
chmod +x /usr/local/bin/cron_test_script.sh

echo "* * * * * /usr/local/bin/cron_test_script.sh" | crontab -
sleep 65
cat /tmp/cron_test_output.log      # Now succeeds, consistently, regardless of invocation context
crontab -r
```

Practice `at` for a genuine one-time deferred task, including the Day 2 safety-net pattern:
```bash
echo "systemctl restart sshd" | at now + 2 minutes
atq                                  # List pending at jobs
at -c $(atq | head -1 | awk '{print $1}')   # Inspect exactly what a specific pending job will run

# The Day 2 safety-net pattern, revisited with full understanding now:
echo "nmcli con up backup-profile" | at now + 5 minutes
# (if a risky network change about to be tested breaks connectivity, this scheduled
#  job fires regardless and restores it — cancel with `atrm <jobnumber>` once confirmed safe)
```

### Interview Questions — with Answers

**1. A script runs perfectly when a user executes it manually from their shell, but silently does nothing (or fails partway) when run from that same user's crontab. What's your default hypothesis, and how do you confirm it quickly?**

My default hypothesis is an environment difference — most commonly, the script relies on something only present in the user's interactive shell environment (a `$PATH` addition from `.bashrc`, an exported environment variable, an alias) that cron's minimal execution environment simply doesn't have, since cron does not source the user's shell configuration files. I'd confirm quickly by checking the script for any use of bare command names without full paths, cross-referencing those against cron's actual (minimal) `$PATH`, and — most directly — temporarily modifying the cron job to redirect ALL output (stdout AND stderr) to a log file (`>> /tmp/debug.log 2>&1`), which often immediately reveals a "command not found" error that was invisible when the job "silently did nothing" from cron's own default behavior of emailing or discarding output depending on configuration.

**2. Why is redirecting a cron job's output to a log file (`>> logfile 2>&1`) considered a best practice, even for jobs that seem to be working fine?**

Cron's default handling of a job's output (traditionally, emailing it to the user via local mail, if configured at all, or silently discarding it if not) means that WITHOUT explicit redirection, you have no reliable, persistent record of what a job actually did or whether it encountered errors — a job that appears to be "working fine" today could start silently failing tomorrow (a changed dependency, a permissions issue, a full disk) with no visible indication anything went wrong, since there's no log to check. Explicit redirection to a log file ensures there's always a concrete, inspectable record of every run's actual output and exit behavior, turning "presumably still working" into something you can actually VERIFY, and giving you the diagnostic data needed immediately if something does eventually go wrong, rather than needing to first figure out how to even START capturing the failure.

**3. What's the difference between placing a cron entry in a user's own `crontab -e` versus a file in `/etc/cron.d/`, and when would you choose the system-wide location instead?**

A user's own `crontab -e` entry is scoped to and runs as that specific user, with a standard 5-field syntax (minute, hour, day, month, weekday, command) — appropriate for personal or application-specific scheduled tasks tied to one particular user account. `/etc/cron.d/` files are system-wide, typically installed/managed by packages or deliberately placed by an administrator for infrastructure-level scheduled tasks, and include an ADDITIONAL user field (since the file itself isn't inherently tied to any one user's crontab) specifying which user each individual job should run as — I'd choose `/etc/cron.d/` for genuinely system-level scheduled tasks that should be version-controlled/deployed as part of system configuration management (rather than living in an individual user's personal, less centrally-managed crontab), or for jobs that legitimately need to run as different users within the same logical scheduling file.

**4. Explain the practical difference between `cron` and `at`, and describe a real scenario (beyond the textbook definition) where `at` is genuinely the better tool.**

`cron` schedules RECURRING jobs on a fixed, repeating schedule; `at` schedules a single, ONE-TIME job to run at a specified future point, after which it's done and doesn't repeat. A real, practical scenario where `at` is genuinely the better fit: the Day 2 "zero-lockout SSH network change" safety pattern — scheduling a one-time rollback command to fire a few minutes after a risky network configuration change, specifically so that if the change breaks connectivity, the pre-scheduled rollback self-heals the situation even if you're locked out; this is a genuinely one-off, single-execution need tied to a specific moment's risk, not a recurring schedule cron is designed for — using cron for this (and remembering to manually remove the cron entry afterward) would be a clumsier, more error-prone way to achieve the same one-time-deferred-action goal `at` is purpose-built for.

**5. A cron job that used to run fine suddenly starts failing after an unrelated system update or change elsewhere on the server. Beyond the classic PATH issue, what OTHER environment-related cause should you consider, given what you've learned across this guide so far?**

Beyond PATH, I'd consider whether the job's dependencies involve anything covered on Day 8 — a clock/time-sync issue (drift affecting a job that's time-sensitive or that authenticates against something with tight time-window tolerances, like Kerberos), or an NFS-mounted path the job reads/writes that's become stale or is hanging (per Day 8's stale-vs-hung distinction) rather than the job's own logic actually being broken. I'd also consider an SELinux context change (Day 7) — if something relabeled a file the job depends on, or if the job started needing to touch a resource with a type it doesn't have policy permission for, that would produce a failure that looks environment-related but traces back to a security-context issue rather than PATH/variables specifically — the broader lesson being that "the job's environment changed" can mean many different things across everything covered this week, not just the classic PATH/shell-config case, and a systematic check across these categories is more reliable than assuming it's always the same familiar cause.

---

## Topic 3: Logrotate — Managing Log Growth Without Manual Intervention

### Quick Review
- `logrotate` runs via a systemd timer (or traditionally cron) on a schedule, checking each configured log file against ROTATION CRITERIA (size, age) and rotating those that qualify.
- Configuration lives in `/etc/logrotate.conf` (global defaults) plus `/etc/logrotate.d/*` (per-application configs, typically installed by the packages that own those logs).
- `postrotate`/`endscript` blocks let you run a command AFTER rotation — commonly signaling the application to reopen its log file handle, since simply renaming a file doesn't make an already-running process start writing to the NEW file automatically.
- `logrotate -d` (debug/dry-run) shows exactly what WOULD happen without actually rotating anything — the correct way to test a new configuration before trusting it in production.

### Quick Learning

The single most important, most-forgotten piece of logrotate configuration is the `postrotate` script, and understanding WHY it's necessary requires understanding what rotation actually does at the filesystem level: renaming/moving the current log file and creating a new, empty one in its place does NOT automatically cause an already-running application to start writing to that new file — the application still has its original file handle open, pointing at what is now the RENAMED (old) file's inode, completely unaware a "new" file now exists at the original path. Without a `postrotate` step telling the application to reopen its log file (commonly via a signal like `SIGHUP`, or a reload command), the application keeps writing to the old, now-renamed file indefinitely — meaning the "new" log file at the expected path stays empty forever, while log data keeps accumulating in a file that's been renamed out of its expected location and may even eventually get deleted by a LATER rotation cycle, silently losing log data.

**Why postrotate is necessary — what rotation does vs. what the running application doesn't know:**
```
  BEFORE rotation:
     /var/log/myapp.log  (inode 12345)  ◄── myapp's process has this
                                              file open via inode 12345,
                                              actively writing to it

  logrotate renames it:
     /var/log/myapp.log.1  (STILL inode 12345)  ◄── myapp's process is
                                                       STILL writing here —
                                                       renaming a file does
                                                       NOT change what an
                                                       already-open file
                                                       handle points to

     /var/log/myapp.log  (NEW, inode 67890, empty)  ◄── this is what NEW
                                                           processes/tools
                                                           looking at this
                                                           PATH would see —
                                                           but myapp itself
                                                           doesn't know this
                                                           new file exists

  WITHOUT postrotate: myapp keeps writing to inode 12345 (now named
  .log.1) forever — /var/log/myapp.log stays empty, and log data is
  accumulating somewhere increasingly hard to find, until an eventual
  further rotation cycle might delete .log.1 (per rotate count
  settings) with myapp STILL WRITING to it, silently losing data

  WITH postrotate (e.g., `systemctl reload myapp` or sending SIGHUP):
     myapp is explicitly told to CLOSE its current file handle and
     REOPEN the log file at the expected path — picking up the NEW,
     empty inode 67890 correctly, resuming normal logging behavior
     at the path everything else expects
```

### Implementation (Learn by Applying)

**Scenario:** Configure logrotate for a custom application's log file, deliberately WITHOUT a postrotate step first, to observe the exact silent-data-loss failure mode described above — then fix it correctly.

```bash
mkdir -p /var/log/myapp
cat > /var/log/myapp/myapp.log << 'EOF'
initial log content
EOF

# Simulate a long-running process continuously writing to this log (in the background, for the lab)
( while true; do date >> /var/log/myapp/myapp.log; sleep 2; done ) &
APP_PID=$!

cat > /etc/logrotate.d/myapp << 'EOF'
/var/log/myapp/myapp.log {
    size 1k
    rotate 3
    missingok
    notifempty
}
EOF
```

Force an immediate rotation test and observe the problem:
```bash
logrotate -f /etc/logrotate.d/myapp

ls -la /var/log/myapp/                  # Shows myapp.log (new, small/empty) AND myapp.log.1 (the renamed original)
sleep 5
tail -3 /var/log/myapp/myapp.log         # Likely still EMPTY or not growing
tail -3 /var/log/myapp/myapp.log.1       # The background process's writes are STILL going HERE — the actual bug in action
```

Fix it correctly with a postrotate step:
```bash
cat > /etc/logrotate.d/myapp << 'EOF'
/var/log/myapp/myapp.log {
    size 1k
    rotate 3
    missingok
    notifempty
    postrotate
        kill -HUP $(cat /tmp/myapp.pid 2>/dev/null) 2>/dev/null || true
    endscript
}
EOF
```
For this lab specifically (a plain background loop, not a real HUP-aware daemon), demonstrate the CONCEPT by simply restarting the writer process after rotation, which is functionally what postrotate accomplishes for a real application:
```bash
kill $APP_PID 2>/dev/null
( while true; do date >> /var/log/myapp/myapp.log; sleep 2; done ) &
sleep 5
tail -3 /var/log/myapp/myapp.log         # NOW growing correctly, at the expected path
```

Test any new logrotate configuration safely before trusting it:
```bash
logrotate -d /etc/logrotate.d/myapp      # Debug/dry-run — shows exactly what WOULD happen, changes nothing
```

Clean up the lab's background process:
```bash
pkill -f "date >> /var/log/myapp"
```

### Interview Questions — with Answers

**1. Explain exactly why a `postrotate` script is often necessary after log rotation, using the actual filesystem-level mechanism — not just "it tells the app to reopen the file."**

Log rotation, at the filesystem level, renames the current log file (the application's ALREADY-OPEN file handle still points to the same underlying inode, now accessible under the new, renamed filename) and creates a NEW, empty file at the original path with a DIFFERENT inode. Because the running application's open file handle references the inode directly, not the path, renaming the file doesn't change what the running process is actually writing to — it continues writing to the same inode, now living under a different filename, completely unaware anything happened, while the "new" file at the expected path remains empty since nothing is writing to IT. A `postrotate` script (commonly signaling the application via `SIGHUP` or an equivalent reload mechanism) is necessary specifically to tell the application to explicitly close its current file handle and reopen the log file at the expected path, which causes it to pick up the NEW inode and resume writing where everything else (monitoring, log shipping, manual inspection) expects to find current log data.

**2. What's the real-world consequence of forgetting a `postrotate` step for an application that DOES need one, beyond "the new log file is just empty"?**

Beyond the immediate symptom of an empty "current" log file, the actual data-loss risk is more serious: the application keeps writing normally to the OLD, now-renamed file (e.g., `myapp.log.1`) indefinitely — and if the logrotate configuration has a `rotate N` count limiting how many old rotated files are kept, a LATER rotation cycle can eventually DELETE that file (per the normal rotation/cleanup policy) while the application is STILL actively writing to it, since logrotate has no way of knowing the application never actually stopped using it. This results in genuine, permanent log data loss — not just temporary inconvenience — and can go unnoticed for a long time specifically because nothing produces an obvious ERROR; the application keeps running and "logging" successfully from its own perspective, the failure is entirely about WHERE that data ends up relative to where everything else expects to find it.

**3. How would you safely test a new or modified logrotate configuration BEFORE trusting it to run automatically in production, without risking actually disrupting real log files?**

`logrotate -d <config-file>` (debug mode) performs a full dry run, showing exactly what actions WOULD be taken — which files would be rotated, what the resulting filenames would be, whether any configured postrotate scripts would run — without actually executing any of it or modifying any real files. This is the correct, safe way to validate a new configuration's logic (rotation criteria, file matching, retention count) matches what's actually intended, catching syntax errors or unexpected matching behavior before the configuration is ever allowed to run for real and potentially rotate (or fail to properly rotate) production log files in an unintended way.

**4. What's the difference between `notifempty` and `missingok` in a logrotate configuration, and why would omitting either one cause a real problem for a specific kind of log file?**

`notifempty` tells logrotate to SKIP rotating a log file if it's currently empty — without this, a log file that genuinely has no new content since the last rotation would still get rotated anyway, needlessly accumulating empty rotated files and cluttering the rotation history with zero actual value. `missingok` tells logrotate not to treat it as an ERROR if the configured log file doesn't exist at all at rotation time — without this, a genuinely missing log file (perhaps an application that isn't currently running, or a log path that only gets created under certain conditions) would cause logrotate to report an error/failure for that specific configuration entry, which could mask OTHER, genuinely important errors in the same logrotate run if you're not carefully distinguishing "expected, harmless missing file" from "an actual problem" in your monitoring of logrotate's own execution results.

**5. A team's application writes logs to a location managed by logrotate, but their monitoring/alerting for "is this application still logging normally" keeps producing false positives right around each rotation window. What's the likely connection to logrotate's mechanics specifically?**

The likely connection is exactly the postrotate/file-handle mechanism from this topic — if the application's postrotate reopening happens with any delay, or if monitoring checks the log file's freshness/recent-write-timestamp precisely during the brief window between the actual rotation event and the application successfully reopening its handle to the new file, the monitoring might observe a brief, entirely normal and expected gap in writes to the CURRENT path and incorrectly interpret that as "the application stopped logging/is unhealthy." I'd investigate whether the monitoring's alerting threshold accounts for this expected, brief rotation-transition gap, and confirm the postrotate mechanism itself is working correctly and reasonably promptly (not introducing an unusually LONG gap beyond what's normal/expected) — distinguishing "monitoring alert threshold too tight for a known, benign rotation-transition gap" from "postrotate is genuinely slow or broken and needs its own investigation" as two different possible fixes for the same observed symptom.

---

## Topic 4: Shell Scripting Basics for Automation

### Quick Review
- `set -euo pipefail` at the top of a script is the baseline defensive habit: exit on any error (`-e`), treat unset variables as errors (`-u`), and fail a pipeline if ANY command in it fails, not just the last one (`pipefail`).
- Idempotency — a script safely re-runnable multiple times with the same end result — is a real production requirement, not a nice-to-have, since automation genuinely does get re-run (retries, re-deployment, manual re-execution after a partial failure).
- Explicit exit codes and proper error handling (checking command success, not assuming it) separate a script that LOOKS like it works from one that's actually safe to run unattended.
- Logging every significant action a script takes (with timestamps) is what makes a script's automated behavior debuggable after the fact, rather than a black box.

### Quick Learning

The gap between "a script that works when I run it once, watching" and "a script safe to run unattended, repeatedly, in production automation" is almost entirely about defensive habits most people learn only after being burned by their absence: not checking whether commands actually succeeded before proceeding, not handling the case where the script gets re-run after a partial previous failure, and not distinguishing between "this script ran" and "this script actually accomplished its intended goal correctly." This is exactly the practical, hands-on-experience-gated knowledge a product-company interviewer is listening for when they ask "tell me about a script you've written for automation" — a script's basic logic being correct is almost assumed; whether it's SAFE and IDEMPOTENT is the actual differentiator.

**Non-idempotent vs. idempotent — the same intent, genuinely different safety:**
```
  NON-IDEMPOTENT (re-running breaks things):        IDEMPOTENT (re-running is safe):
  ──────────────────────────────────────────         ─────────────────────────────────
  mkdir /app/data                                    mkdir -p /app/data
  (FAILS on second run — directory                   (succeeds every time, whether
   already exists, script errors out                  the directory already existed
   or worse, continues past the error                  or not — this is the WHOLE
   if -e isn't set, in an unknown state)                point of the -p flag)

  useradd appuser                                     id appuser &>/dev/null || useradd appuser
  (FAILS on second run — user already                (checks FIRST whether the user
   exists; without proper error handling,              already exists, only creates
   script may proceed as if this step                  it if genuinely needed —
   succeeded when it actually errored)                  re-running is a safe no-op)

  echo "some_setting=value" >> /etc/app.conf          grep -q "^some_setting=" /etc/app.conf || \
  (appends AGAIN every re-run — after                    echo "some_setting=value" >> /etc/app.conf
   3 re-runs, the file has the SAME                  (checks if the setting is ALREADY
   line duplicated 3 times, potentially                present before appending — avoids
   causing genuinely broken behavior                   duplicate entries on re-run)
   depending on how the app parses it)

  Idempotent scripts CHECK CURRENT STATE before
  acting, rather than blindly performing an action
  and assuming a clean, first-time starting point
  every single run.
```

### Implementation (Learn by Applying)

**Scenario:** Write a real automation script — provisioning a new application user and directory structure — first the naive, non-idempotent way (and prove it breaks on re-run), then rebuild it defensively and idempotently, with proper error handling and logging.

```bash
cat > /usr/local/bin/provision_app_naive.sh << 'EOF'
#!/bin/bash
useradd appuser
mkdir /opt/appdata
chown appuser:appuser /opt/appdata
echo "APP_ENV=production" >> /etc/app.conf
echo "Provisioning complete"
EOF
chmod +x /usr/local/bin/provision_app_naive.sh

/usr/local/bin/provision_app_naive.sh          # First run — succeeds
/usr/local/bin/provision_app_naive.sh; echo "exit code: $?"   # Second run — observe the failures
cat /etc/app.conf                                # Note the DUPLICATED line — a real, growing problem on repeated runs
```

Rebuild it properly — defensive, idempotent, logged:
```bash
cat > /usr/local/bin/provision_app.sh << 'SCRIPT'
#!/bin/bash
set -euo pipefail

LOG=/var/log/provision_app.log
log() { echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a "$LOG"; }

log "Starting provisioning"

if id appuser &>/dev/null; then
    log "User appuser already exists, skipping creation"
else
    useradd appuser
    log "Created user appuser"
fi

mkdir -p /opt/appdata
log "Ensured /opt/appdata exists"

chown appuser:appuser /opt/appdata
log "Set ownership on /opt/appdata"

if grep -q "^APP_ENV=" /etc/app.conf 2>/dev/null; then
    log "APP_ENV already configured, skipping"
else
    echo "APP_ENV=production" >> /etc/app.conf
    log "Added APP_ENV setting"
fi

log "Provisioning complete"
SCRIPT
chmod +x /usr/local/bin/provision_app.sh

# Clean up the naive script's mess first for a fair test
userdel appuser 2>/dev/null; rm -rf /opt/appdata; rm -f /etc/app.conf

/usr/local/bin/provision_app.sh                 # First run
/usr/local/bin/provision_app.sh                 # Second run — should complete cleanly, logging what it skipped
/usr/local/bin/provision_app.sh                 # Third run — same, provably safe to re-run indefinitely
cat /etc/app.conf                                 # Confirm: exactly ONE line, not duplicated across runs
cat /var/log/provision_app.log                    # Full audit trail of every run's actual decisions
```

### Interview Questions — with Answers

**1. Explain what `set -euo pipefail` actually does, each flag individually, and describe a real bug each one specifically prevents.**

`-e` causes the script to exit immediately if any command returns a non-zero (failure) exit status, rather than the default bash behavior of continuing to the next line regardless — without it, a script can proceed past a genuinely failed step (a failed `mkdir`, a failed download) and continue executing SUBSEQUENT commands that assume that step succeeded, potentially causing confusing downstream failures or, worse, silent incorrect behavior. `-u` treats the use of any UNSET variable as an error rather than silently substituting an empty string — without it, a typo in a variable name (referencing `$FOOO` when you meant `$FOO`) silently evaluates to an empty string rather than immediately surfacing as an obvious bug, potentially causing a command to run with a missing/empty argument it needed. `pipefail` ensures that if ANY command within a pipeline fails (not just the LAST one), the overall pipeline's exit status reflects that failure — without it, `somecommand_that_fails | grep something` reports success (based on `grep`'s exit status alone) even though `somecommand_that_fails` genuinely failed, masking a real failure behind an unrelated, successful-looking final command in the pipe.

**2. What does it mean for a script to be "idempotent," and why does this matter for real production automation specifically, beyond "it's good practice"?**

An idempotent script produces the same correct end state regardless of how many times it's run, safely handling the case where some or all of its actions have already been performed by a previous run — re-running it doesn't error out, duplicate work, or leave the system in a worse or inconsistent state. This matters for real production automation because scripts genuinely DO get re-run in practice: a deployment pipeline might retry a failed step, an admin might manually re-execute a provisioning script after investigating and fixing an unrelated issue that caused a partial failure partway through, or a configuration management tool might run the same script repeatedly by design (convergence-style tooling). A non-idempotent script that assumes it's always running against a clean, first-time state will genuinely break or produce incorrect/duplicated results in exactly these common, realistic re-run scenarios — idempotency isn't an abstract best practice, it's a concrete requirement for a script to be trustworthy in the actual conditions production automation runs under.

**3. A script appears to run successfully (no error messages, exits with status 0) but doesn't actually accomplish what it was supposed to. What kind of scripting mistake commonly causes this specific gap between "ran without error" and "actually worked"?**

This commonly happens when a script doesn't properly CHECK the success/failure of individual commands or the actual RESULT of an operation, only relying on the overall script not crashing outright — for example, a command that fails but whose failure doesn't cause the overall script to exit non-zero (perhaps because `set -e` wasn't used, or because the failing command was inside a construct that swallows its exit status), combined with subsequent commands that don't themselves fail even though they're now operating on incorrect/missing prerequisites, can result in a script that reports overall success while having silently skipped or failed at its actual core task. The fix is exactly the defensive pattern from the lab: explicit `set -euo pipefail`, explicit checks of command results where automatic exit-on-failure isn't sufficient (like checking a downloaded file's checksum, not just whether the download command itself returned 0), and logging enough detail about what the script actually DID at each step to allow after-the-fact verification that it genuinely accomplished its intended goal, not just that it didn't crash.

**4. Why is logging every significant action a script takes (with timestamps) considered important for production automation scripts specifically, beyond general good practice?**

Production automation scripts typically run UNATTENDED — nobody is watching the terminal output in real time the way you would during manual, interactive execution — meaning if something goes wrong, or if you later need to understand exactly what a script did during a specific run (for an audit, an incident investigation, or simply confirming it behaved as expected), the ONLY record available is whatever the script itself explicitly logged; there's no human memory of watching it run to fall back on. Timestamped logging specifically also enables correlating a script's actions against other systems' logs (connecting back to Day 3's journalctl cross-boot correlation and Day 8's clock-drift-affects-log-correlation lessons) — understanding not just WHAT a script did, but exactly WHEN, relative to other events happening on the system or across other systems, which is often essential for understanding cause and effect during a real investigation.

**5. You're reviewing a colleague's automation script before it's approved for production use. Beyond checking that the core logic is correct, what specific things would you look for regarding safety and idempotency?**

I'd check whether every state-changing operation (creating a user, a directory, a file, a config entry) first checks whether that state already exists before acting, rather than blindly performing the action and assuming a clean starting point — exactly the naive-vs-idempotent distinction from the lab. I'd check for `set -euo pipefail` (or equivalent explicit error handling if the script's structure genuinely needs different handling for specific expected-to-possibly-fail commands) rather than a script that silently continues past failures. I'd check for adequate, timestamped logging of significant actions, sufficient to reconstruct what happened during any given run after the fact. And I'd specifically ask (or test myself) what happens if the script is run a SECOND time immediately after a successful first run — if that's not something the author has explicitly considered and tested, it's a strong signal the script isn't actually production-ready yet, regardless of how clean its core logic looks on a single, first, successful run.

---

## Topic 5 (Bonus — Advanced): sysctl — Kernel Parameter Tuning

### Quick Review
- `/proc/sys/` is the LIVE, in-memory representation of every tunable kernel parameter — `sysctl` is simply the tool for reading and writing these values conveniently.
- `sysctl -w param=value` changes a parameter IMMEDIATELY but only for the CURRENT boot session; persistent changes require writing to a file under `/etc/sysctl.d/` and either rebooting or running `sysctl -p <file>`.
- A handful of parameters come up constantly in real production tuning conversations: `net.core.somaxconn` (connection backlog queue size), `vm.swappiness` (how aggressively the kernel swaps), `fs.file-max` (system-wide open file descriptor limit), `net.ipv4.ip_forward` (from Day 6's NAT gateway topic).
- Tuning a sysctl value without understanding WHAT workload problem it's actually solving is a common way to introduce a poorly-understood, hard-to-explain-later production change — every tuning decision should be traceable to an actual, observed need.

### Quick Learning

sysctl tuning has the same trap as most "here's a knob you can turn" topics: candidates who've memorized a list of "common" sysctl parameters and their typical values, without understanding the actual PROBLEM each one solves, give shallow, unconvincing answers under real interview pressure ("why did you set THIS value specifically, for THIS workload"). The deeper, correct framing: every sysctl tuning decision should trace back to an observed, specific symptom or requirement (a connection-backlog related dropped-connection pattern, an out-of-memory/swap-thrashing pattern from Day 10's performance topic, a "too many open files" error) — not applied preemptively as generic "best practice hardening" without a concrete reason tied to the actual workload in front of you.

**Runtime vs. persistent — the same trap as every other "live change vs config file" pattern this week:**
```
  sysctl -w net.core.somaxconn=1024
  ─────────────────────────────────
  Takes effect IMMEDIATELY — the running kernel's actual
  behavior changes right now, live

  ...but written ONLY to /proc/sys/net/core/somaxconn's
  current in-memory value — NOT to any persistent config file

  Next REBOOT: reverts to whatever the DEFAULT (or whatever's
  actually configured in /etc/sysctl.d/) was, SILENTLY —
  exactly the same "tested live, forgot to persist, reverted
  at the next reboot/reload" trap that showed up with firewalld
  zones (Day 6) and NFS/idmapd domain config (Day 8)

  Persistent fix:
     echo "net.core.somaxconn = 1024" >> /etc/sysctl.d/99-tuning.conf
     sysctl -p /etc/sysctl.d/99-tuning.conf    <- applies AND confirms
                                                   the persistent file's
                                                   syntax is correct,
                                                   without needing an
                                                   actual reboot to test it
```

### Implementation (Learn by Applying)

**Scenario:** A web application server is experiencing dropped connections under load, specifically that a `ss -s` check reveals a pattern consistent with the connection backlog queue filling up before the application can accept new connections fast enough. Diagnose the need, apply the correct sysctl tuning, and persist it correctly.

```bash
# Check the CURRENT value and understand what it actually controls
sysctl net.core.somaxconn
# This controls the maximum length of the queue for FULLY ESTABLISHED sockets waiting to be
# accept()'d by the application — a queue that's too small under high connection-rate load
# can cause the kernel to start REFUSING new connections even though the server has capacity
# to handle them, simply because they can't get INTO the backlog queue fast enough

ss -s                                  # Look for signs consistent with backlog-related issues under load
```

Apply the tuning, correctly, both live and persistently:
```bash
sysctl -w net.core.somaxconn=4096      # Immediate effect, for testing NOW under current load conditions

echo "net.core.somaxconn = 4096" > /etc/sysctl.d/99-webapp-tuning.conf
sysctl -p /etc/sysctl.d/99-webapp-tuning.conf    # Validates syntax AND confirms it applies correctly
sysctl net.core.somaxconn                          # Confirm the persistent value matches the live one
```

Explore other genuinely common, real production sysctls — always check current value and understand WHY before changing:
```bash
sysctl vm.swappiness
# Controls how aggressively the kernel swaps memory pages to disk vs. reclaiming page cache
# instead — a LOWER value makes the kernel prefer reclaiming cache over swapping actual
# process memory, generally preferred on servers (vs. desktop systems) where swapping
# active application memory under light memory pressure is usually worse for performance
# than shrinking filesystem cache

sysctl fs.file-max
# System-wide MAXIMUM number of file descriptors the kernel will allow to be open
# simultaneously across ALL processes — relevant when a high-connection-count service
# (many simultaneous sockets, each consuming a file descriptor) hits a wall that looks
# like "too many open files" but is actually a SYSTEM-WIDE ceiling, not a per-process
# ulimit issue (a genuinely different, commonly confused root cause)
ulimit -n                               # Compare against the PER-PROCESS/PER-USER limit — a DIFFERENT,
                                          # separately-configured ceiling from the system-wide fs.file-max
```

### Interview Questions — with Answers

**1. A team wants to proactively tune `vm.swappiness` and several other "commonly recommended" sysctl parameters on a new server, before any actual performance issue has been observed. How would you respond to this request?**

I'd push back on tuning purely preemptively without a concrete, observed reason — every sysctl change should trace back to an actual, specific workload characteristic or symptom you're addressing, not a generic "best practice" list applied blindly; a value that's genuinely optimal for one workload can be actively WRONG for a different one (a memory-intensive batch-processing workload has different optimal swappiness behavior than a low-latency web-serving workload, for instance). I'd propose instead: establish proper baseline monitoring first (tying back to Day 10's upcoming performance topic), let the server run under REAL, representative load, and make specific, justified tuning decisions based on actually-observed behavior and bottlenecks — documenting WHY each specific value was chosen, tied to a specific observed need — rather than applying a generic tuning checklist that might not even be appropriate for this specific workload, and that would be much harder to explain or justify later ("why is swappiness set to 10 on this box?" should have a real, workload-specific answer, not "it was on a list somewhere").

**2. What's the practical difference between `fs.file-max` and the per-user/per-process `ulimit -n`, and why does confusing the two lead to incorrect troubleshooting of a "too many open files" error?**

`fs.file-max` is a SYSTEM-WIDE ceiling on the total number of file descriptors that can be open simultaneously across EVERY process on the entire system combined. `ulimit -n` (and the underlying `/etc/security/limits.conf` configuration) is a PER-PROCESS or PER-USER ceiling, independently limiting how many file descriptors any SINGLE process/user session can have open, regardless of how much headroom remains in the system-wide total. Confusing the two leads to incorrect troubleshooting because "too many open files" could stem from EITHER limit being hit, and the fix for each is completely different — raising `fs.file-max` does nothing to help a process that's hitting its OWN `ulimit -n` ceiling (which is far below the system-wide max), and raising a user's `ulimit` does nothing if the SYSTEM-WIDE `fs.file-max` is what's actually being exhausted (perhaps by many DIFFERENT processes/users collectively, none of which individually hit their own per-process limit) — correctly diagnosing which ceiling is actually the bottleneck requires checking both, not assuming based on the error message alone which one is relevant.

**3. Why does `sysctl -w` alone leave a production server vulnerable to an unexpected, silent configuration reversion — and describe the specific operational scenario where this actually bites someone.**

`sysctl -w` changes the LIVE, in-memory kernel parameter value immediately, but doesn't write that change to any persistent configuration file — it only exists in the running kernel's current state, which is entirely reset back to whatever's defined in `/etc/sysctl.d/` (or kernel defaults, if nothing's configured there) at the next boot. The specific scenario: an admin diagnoses a real production issue, applies a `sysctl -w` fix live, confirms it resolves the immediate problem, and moves on — without also writing the equivalent line to a persistent `/etc/sysctl.d/` file — and weeks or months later, a routine reboot (for patching, maintenance, or an unplanned restart) silently reverts that tuning back to its default, and the EXACT SAME originally-diagnosed issue reappears, confusing whoever's investigating it THIS time, since from their perspective nothing was recently changed — this is precisely the same category of "live change never persisted" trap that showed up with firewalld zones on Day 6 and NFS idmapd configuration on Day 8, now appearing again at the kernel-tuning layer.

**4. How would you verify that a sysctl parameter you've configured in `/etc/sysctl.d/` will actually apply correctly and without syntax errors, WITHOUT needing to reboot the server to find out?**

`sysctl -p <specific-file-path>` (or `sysctl --system` to apply ALL configured sysctl.d files at once) explicitly re-reads and applies the specified persistent configuration file immediately, both confirming the file's syntax is valid (reporting an error if it's malformed) AND actually applying the values live in the same operation — this gives you confidence the persistent configuration is correct and will apply successfully at the NEXT boot too (since it's the same mechanism/files that boot-time sysctl processing uses), without needing to actually reboot the server just to test whether a config file has a typo or syntax mistake in it.

**5. Beyond tuning individual sysctl values in isolation, how would you approach BUILDING a case for a specific sysctl change to present to a team or during a code/config review — what evidence would make the change credible and well-justified rather than looking like guesswork?**

I'd present the actual observed symptom or bottleneck that motivated the investigation in the first place (specific metrics, error messages, or monitoring data showing the actual problem — e.g., connection drops correlating with backlog exhaustion, or actual swap activity correlating with performance degradation), explain the SPECIFIC mechanism by which the proposed sysctl change addresses that observed symptom (not just "this is a commonly recommended value"), and ideally show before/after evidence from a controlled test (a staging environment under representative load, or the change applied live with before/after monitoring data) confirming the change actually resolved the observed issue without introducing a new, different problem. This evidence-based, symptom-to-mechanism-to-verified-fix chain is what separates a credible, defensible tuning decision from something that looks like it was copied from a generic "top 10 sysctl tunings" list without genuine understanding of why THIS specific server, under THIS specific workload, actually needed THIS specific change.

---

**End of Day 9.** You should now be able to perform genuine root password/access recovery with full understanding of WHY each step matters (not just memorized steps), diagnose and fix the classic cron environment-mismatch bug rather than being confused by it, understand log rotation's postrotate mechanism deeply enough to explain the exact filesystem-level reason it's necessary, write automation scripts with real production-grade defensive habits (idempotency, proper error handling, meaningful logging) rather than scripts that merely "work once while I'm watching," and — as the advanced capstone — approach sysctl kernel tuning as an evidence-based, symptom-driven practice rather than applying a generic checklist of commonly-recommended values.

Proceed to **Day 10 — Performance & Package Management** next.
