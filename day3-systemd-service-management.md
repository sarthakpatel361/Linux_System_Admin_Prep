# Day 3 — Systemd & Service Management (Full Study Edition, with Answers)

**Estimated time: 5–6 hours**

Systemd is the layer where "the server is up" and "the service is actually healthy" stop being the same question. Product-company interviewers use systemd questions to test something specific: can you reason about *why* a unit started, failed, or came up in the wrong order — not just recite `systemctl start/stop/enable`. This day goes deeper than Day 1 and Day 2: six topics, covering the dependency graph itself, custom unit authoring, the mask/enable distinction and timer units, a full journalctl deep-dive, boot-failure triage, and — as the advanced capstone — how systemd's cgroup integration lets you enforce resource limits per service, which is where systemd starts overlapping directly with what you'd call "performance and capacity work" elsewhere in this guide.

---

## Topic 1: Units, Targets & the Boot Dependency Graph

### Quick Review
- A **unit** is any manageable systemd object — `.service`, `.target`, `.socket`, `.timer`, `.mount`, etc.
- A **target** is a grouping of units representing a system state (e.g., `multi-user.target` ≈ old runlevel 3).
- `After=`/`Before=` control **ordering only**; `Wants=`/`Requires=` control **pulling-in** (with different failure tolerance).
- `systemctl list-dependencies` and `systemd-analyze` let you see the actual dependency graph rather than guessing.

### Quick Learning

Systemd boots the system by resolving a dependency graph of units, not by running scripts in a fixed numbered order the way SysV init did. Every unit can declare relationships to other units — some purely about *ordering* (`After=`, `Before=`), some about *pulling in* other units as part of activating this one (`Wants=`, `Requires=`), and the two dimensions are independent: a unit can be ordered after another without depending on it, or depend on another without caring about ordering. The single most-confused pair in interviews is `After=` versus `Requires=` — `After=` says nothing about whether the dependency actually gets started at all, it only affects sequencing *if* it's going to start anyway for some other reason.

**Ordering vs. pulling-in — two independent dimensions:**
```
                    Requires=  (pulls dependency in; THIS unit fails
                                 if the dependency fails)
                        │
  myapp.service ────────┤
                        │
                    Wants=     (pulls dependency in; tolerates its
                                 failure — THIS unit still starts)

  ─────────────────────────────────────────────────────────────
                    After=     (ordering ONLY — does not pull
                                 anything in by itself)
                        │
  myapp.service ────────┤
                        │
                    Before=    (ordering ONLY — the opposite direction)

  A unit with ONLY "After=network-online.target" and no Wants=/Requires=
  will start in whatever order network-online.target happens to reach,
  but if nothing ELSE pulls network-online.target in, it might never
  start at all — After= alone never guarantees the dependency runs.
```

**The boot sequence as a target chain:**
```
  default.target (usually -> multi-user.target or graphical.target)
        │
        ▼
  multi-user.target
        │  Wants=  (pulls in most system services)
        ▼
  ┌─────────────┬─────────────┬─────────────┐
  │ sshd.service  │ network.target│ myapp.service │   ...many more
  └─────────────┴─────────────┴─────────────┘
        │
        ▼
  basic.target ──▶ sysinit.target ──▶ (early boot: mounts, udev, etc.)
```

### Implementation (Learn by Applying)

**Scenario:** You need to understand exactly why a service is starting later than expected in the boot sequence, and prove your understanding of the dependency graph rather than assuming.

```bash
# See what a target actually pulls in
systemctl list-dependencies multi-user.target

# See a specific unit's own dependency relationships
systemctl list-dependencies sshd.service --all
systemctl show sshd.service -p After -p Wants -p Requires -p WantedBy

# See the ACTUAL boot-time ordering and timing, not the theoretical graph
systemd-analyze blame                 # Which units took the longest to start
systemd-analyze critical-chain        # The longest dependency CHAIN that determined total boot time
systemd-analyze plot > /tmp/boot.svg  # Visual boot timeline (view via scp/browser if graphical access available)
```

Now build a unit that intentionally demonstrates the After-without-Wants trap:
```bash
cat > /etc/systemd/system/latecheck.service << 'EOF'
[Unit]
Description=Demonstrates After= without Wants=
After=network-online.target

[Service]
ExecStart=/bin/sh -c 'echo "started at $(date)" >> /tmp/latecheck.log'
Type=oneshot

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now latecheck.service
systemctl status latecheck.service
cat /tmp/latecheck.log
```
Note that this unit has no `Wants=network-online.target` — if nothing else in the system happens to pull that target in, `latecheck.service` will still run (since `multi-user.target` wants it directly), but with no guaranteed ordering relative to network readiness beyond "after network-online.target IF it runs." Add `Wants=network-online.target` alongside the existing `After=` and re-test to see the difference in `systemd-analyze critical-chain` output.

### Interview Questions — with Answers

**1. What's the practical difference between `After=` and `Requires=`, and why does confusing them cause real production bugs?**

`After=` is purely about the *order* in which two units start relative to each other, if they're both going to start anyway for some other reason — it creates zero obligation for the dependency to actually run. `Requires=` actually pulls the dependency in as part of activating this unit, and critically, if the dependency fails, this unit fails too (a hard coupling). The real bug this causes: someone writes `After=postgresql.service` on an application unit assuming that guarantees postgresql will be running first, but if nothing else in the system happens to start postgresql, the application unit starts anyway (with nothing to be "after," effectively) — and then fails at runtime trying to connect to a database that was never started, in a way that looks like a flaky application bug rather than the actual systemd dependency-declaration mistake.

**2. Explain the difference between `Wants=` and `Requires=` — walk me through a concrete scenario where using the wrong one causes a cascading failure.**

`Wants=` pulls in the dependency but tolerates its failure — if the wanted unit fails to start, this unit still proceeds normally. `Requires=` creates a hard dependency — if the required unit fails, this unit fails too, and depending on further relationships, that failure can cascade upward to whatever wanted or required THIS unit. Concrete scenario: if a non-critical logging/metrics-shipping sidecar service is declared with `Requires=` by a critical application unit, and that sidecar has a transient failure (bad config push, brief network blip), the critical application goes down with it — purely because of an overly strict dependency declaration on something that should have been a soft `Wants=` relationship, since the app could have functioned (perhaps degraded, but functioning) without the sidecar.

**3. A service seems to start fine sometimes and fail intermittently right after boot, but works perfectly if you manually restart it a minute later. What's your hypothesis, and what commands do you check first?**

My first hypothesis is a missing or insufficient dependency declaration — the service is likely starting before something it actually needs (network fully up, a database, a mounted filesystem) is genuinely ready, and the "intermittent" nature comes from boot-time race conditions where that dependency sometimes happens to be ready in time and sometimes isn't. I'd check `systemctl status <service>` right after a failed boot to see the actual failure reason, `journalctl -u <service> -b` to see what it logged at the failure moment, and `systemctl list-dependencies <service> --all` plus `systemd-analyze critical-chain <service>.service` to see exactly what it's ordered/depending on versus what it probably actually needs at runtime — a mismatch between the two is almost always the root cause of this exact symptom.

**4. What's the difference between `WantedBy=` in a unit's `[Install]` section and `Wants=` in its `[Unit]` section — they sound similar, what's actually different?**

`Wants=` in `[Unit]` describes what THIS unit wants to pull in when IT starts — it's about this unit's own dependencies. `WantedBy=` in `[Install]` is the inverse relationship, declared from the *other* direction: it says "when the named target starts, it should want to pull ME in" — this is what `systemctl enable` actually acts on, creating a symlink so that the target listed in `WantedBy=` will activate this unit when that target itself starts. They're two sides of the same relationship type, just declared from opposite ends: a unit says what it needs via `Wants=`, and separately declares how it should be hooked into the boot sequence via `WantedBy=`.

**5. `systemd-analyze critical-chain` shows a specific unit taking a long time and being on the critical path to reaching `multi-user.target`. What does "critical path" actually mean here, and how would you use that information to actually improve boot time?**

The critical path is the longest chain of ordering dependencies that determines the minimum possible time to reach the target state — even if other units start faster or in parallel, boot time can't be shorter than however long this specific chain takes, because each unit in it is waiting on the previous one. Knowing this, the only way to meaningfully improve boot time is to address something on THIS specific chain — either making the slow unit itself start faster (investigate why it's slow — waiting on a network call, a slow disk check, etc.), or reconsidering whether it genuinely needs to be ordered where it is (could it start in parallel with something else instead of strictly after it, if the ordering dependency was overly conservative). Speeding up units NOT on the critical path has zero effect on total boot time, since they were never the bottleneck to begin with — this is the key insight `critical-chain` gives you over `blame`, which just shows slow units in isolation without showing whether they're actually on the path that matters.

---

## Topic 2: systemctl Service Lifecycle & Custom Unit Authoring

### Quick Review
- Never edit vendor `.service` files directly — use `systemctl edit` to create a drop-in override.
- `systemctl daemon-reload` is mandatory after any unit file change, or systemd keeps using the stale definition.
- `Restart=`/`RestartSec=` define auto-recovery behavior for a crashing service.
- `Type=` (simple/oneshot/forking/notify) tells systemd how to know the service actually started successfully.

### Quick Learning

Writing a correct custom unit file is less about syntax memorization and more about correctly answering one question for each directive: "does systemd need to know this to manage the service's lifecycle correctly?" `Type=` in particular is the field people get wrong most often — it tells systemd how to determine "this service has successfully started," which directly affects whether dependent units wait correctly and whether `systemctl start` returns success/failure accurately.

**Drop-in overrides vs. editing the vendor file directly:**
```
  WRONG                                    RIGHT
  ─────                                    ─────
  vi /usr/lib/systemd/system/              systemctl edit myapp.service
     sshd.service   (edit directly)              │
        │                                         ▼
        ▼                                 Creates:
  Package update/reinstall               /etc/systemd/system/
  OVERWRITES your changes                   myapp.service.d/override.conf
  silently — no warning, just gone              │
                                                 ▼
                                        Survives package updates —
                                        systemd merges your drop-in
                                        with the vendor unit at
                                        runtime, vendor file untouched
```

### Implementation (Learn by Applying)

**Scenario:** Build a custom application service that restarts automatically on failure, waits properly for network availability, and runs as a dedicated non-root user — then safely customize an EXISTING vendor unit's behavior using a drop-in, without ever touching the vendor's original file.

```bash
useradd -r -s /sbin/nologin appuser

cat > /etc/systemd/system/myapp.service << 'EOF'
[Unit]
Description=My Application
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/bin/myapp --config /etc/myapp/config.yml
User=appuser
Restart=on-failure
RestartSec=5
StartLimitIntervalSec=60
StartLimitBurst=3

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now myapp.service
systemctl status myapp.service
```
`StartLimitIntervalSec=60` + `StartLimitBurst=3` together mean: if this service fails and auto-restarts 3 times within any 60-second window, systemd stops trying and marks it failed rather than restart-looping forever — a real production safeguard against a crash-loop hammering resources.

Now customize an existing vendor unit (e.g., increase `sshd`'s restart resilience) via a drop-in, the correct way:
```bash
systemctl edit sshd.service
```
This opens an editor for `/etc/systemd/system/sshd.service.d/override.conf` — add:
```ini
[Service]
Restart=always
RestartSec=3
```
```bash
systemctl daemon-reload
systemctl cat sshd.service          # Shows the MERGED result: vendor unit + your drop-in together
```

### Interview Questions — with Answers

**1. Why does Red Hat/systemd strongly discourage editing vendor unit files directly, and what actually happens the next time that package updates?**

Vendor unit files live under `/usr/lib/systemd/system/` and are owned by the RPM package that provides the service — when that package updates, RPM overwrites the file with the new vendor version, silently discarding any manual edits with no warning or backup of your changes. Using `systemctl edit` instead creates a drop-in override file under `/etc/systemd/system/<unit>.d/`, which is entirely separate from the package-managed file — an update to the vendor unit doesn't touch this directory at all, so your customization survives package updates indefinitely, and systemd merges both files together at runtime automatically.

**2. What does `systemctl daemon-reload` actually do, and what specific symptom would you see if someone edited a unit file but forgot to run it?**

`daemon-reload` tells systemd to re-parse all unit files from disk and rebuild its in-memory representation of the dependency graph and unit definitions — systemd doesn't watch unit files for changes automatically, so without this step, it keeps operating on the OLD, cached definition even though the file on disk has changed. The specific symptom: you edit a unit file (say, changing `ExecStart` or adding a new `Restart=` policy), run `systemctl restart myapp`, and the service behaves exactly as it did BEFORE your edit — no error, just silently using the stale configuration, which is a genuinely confusing debugging experience if you don't already know to check for a missing `daemon-reload`.

**3. Explain what `StartLimitIntervalSec` and `StartLimitBurst` do together, and why omitting them is a real production risk for a service prone to crashing.**

Together they define a rate limit on automatic restarts: if the service fails and is restarted more than `StartLimitBurst` times within any `StartLimitIntervalSec` window, systemd gives up and marks the unit as failed rather than continuing to restart it indefinitely. Omitting them (or setting them too loosely) means a service that enters a genuine crash loop — failing immediately every time it starts, perhaps due to a bad config or a dependency that's actually down — will restart-loop essentially forever, consuming CPU/logging resources on every failed attempt and potentially masking the real underlying problem behind a flood of repetitive log noise, instead of failing cleanly and visibly so someone actually investigates.

**4. What's the practical difference between `Type=simple` and `Type=forking`, and what goes wrong if you pick the wrong one for how your application actually behaves?**

`Type=simple` tells systemd that the process launched by `ExecStart` IS the main service process — systemd considers the service "started" the moment that process is running, and monitors that exact PID for the service's lifetime. `Type=forking` is for traditional daemons that fork into the background and exit the original parent process — systemd needs to track the actual long-running child, typically via a PID file, rather than the process it directly launched (which exits quickly). Picking `simple` for an application that actually forks means systemd thinks the service died the moment the parent process exits (even though a legitimate child is still running fine) — leading to systemd reporting the service as failed/inactive despite it genuinely running, and any `Restart=` policy potentially firing incorrectly and launching duplicate instances.

**5. You need a systemd service to wait until a specific application-level condition is true (not just "the process started") before dependent services are allowed to start. How would you approach this?**

I'd use `Type=notify` combined with the application actually implementing the `sd_notify()` protocol (or a wrapper that does) — this lets the service itself explicitly tell systemd "I am now ready" at the exact moment it's genuinely prepared to serve traffic (e.g., after finishing initialization, warming a cache, or establishing a database connection), rather than systemd assuming readiness the instant the process merely launches. Dependent units ordered `After=` this service will then correctly wait for that explicit readiness signal rather than racing ahead the moment the process ID exists but the application isn't actually ready yet — this is the correct, systemd-native solution to the exact race-condition class of problem that plain `Type=simple` can't express.

---

## Topic 3: Masking, Enabling, and Socket/Timer Units

### Quick Review
- `disable` removes autostart but allows manual `start`; `mask` symlinks to `/dev/null` — **cannot** be started even manually until unmasked.
- **Socket units** enable socket/on-demand activation — the service starts only when a connection actually arrives.
- **Timer units** are systemd's modern replacement for cron, with richer dependency integration and log visibility via `journalctl`.
- `systemctl list-timers` shows scheduled timers with next/last run times, unlike raw crontab.

### Quick Learning

`disable` and `mask` solve genuinely different problems and interviewers use this pair specifically to check you're not just pattern-matching "disable = turn off." `disable` is for "don't start this automatically, but I might still want to run it manually sometimes." `mask` is for "this must never run, period, even if someone tries to start it by hand or another unit tries to pull it in as a dependency" — a much stronger guarantee, useful for genuinely dangerous or conflicting services (e.g., masking a legacy service that conflicts with a replacement you've deployed).

**disable vs mask — what each actually blocks:**
```
                          Can systemctl        Can another unit's
                          start it manually?    Wants=/Requires= start it?
  ──────────────────────────────────────────────────────────────────────
  systemctl disable X       YES  ✔                YES (if pulled in)  ✔
  (just removes autostart
   symlink from targets)

  systemctl mask X          NO   ✘                NO   ✘
  (symlinks unit file        "Failed to start X.service:
   itself to /dev/null)       Unit X.service is masked."
```

### Implementation (Learn by Applying)

**Scenario:** Replace a legacy time-sync daemon with `chronyd`, ensuring the legacy one can genuinely never start again even by accident — then build a systemd timer to replace a cron job, with proper logging visibility that cron never gave you.

```bash
systemctl disable --now ntpd 2>/dev/null || true    # disable first (if present)
systemctl mask ntpd                                    # then mask — belt and suspenders
systemctl status ntpd                                  # confirm it shows "masked"
systemctl start ntpd                                   # confirm this now genuinely fails
```

Now build a timer-based replacement for a cron job that runs a cleanup script nightly:
```bash
cat > /etc/systemd/system/nightly-cleanup.service << 'EOF'
[Unit]
Description=Nightly cleanup job

[Service]
Type=oneshot
ExecStart=/usr/local/bin/cleanup.sh
EOF

cat > /etc/systemd/system/nightly-cleanup.timer << 'EOF'
[Unit]
Description=Run nightly-cleanup.service daily at 2am

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true
Unit=nightly-cleanup.service

[Install]
WantedBy=timers.target
EOF

systemctl daemon-reload
systemctl enable --now nightly-cleanup.timer
systemctl list-timers nightly-cleanup.timer

# Verify a run and check its logs — this is the part cron never gave you cleanly
systemctl start nightly-cleanup.service   # Trigger manually to test without waiting for 2am
journalctl -u nightly-cleanup.service
```
`Persistent=true` matters operationally: if the server was down at 2am (maintenance, outage), the timer fires the job as soon as the system is back up instead of silently skipping that day's run entirely — plain cron has no equivalent concept.

### Interview Questions — with Answers

**1. Explain the exact difference between `disable` and `mask`, and describe a real scenario where using `disable` when you should have used `mask` caused a production incident.**

`disable` removes the autostart symlink so the unit won't start automatically at boot, but it can still be started manually or pulled in as a dependency by another unit's `Wants=`/`Requires=`. `mask` symlinks the unit file itself to `/dev/null`, making it impossible to start under any circumstance — manual or dependency-triggered — until explicitly unmasked. A real scenario: a team disables a legacy service they've replaced, but another, unrelated unit elsewhere in the system still declares `Wants=legacy-service.service` as a dependency — because it's merely disabled (not masked), that dependency relationship still successfully starts it, silently bringing the legacy service back online and potentially conflicting with its replacement (port binding conflicts, competing for the same resource) in a way that looks mysterious until someone realizes disable doesn't block dependency-triggered starts.

**2. What's the practical advantage of a systemd timer over a traditional cron job, beyond "it's the newer way to do it"?**

Concretely: timers integrate with the full systemd dependency graph, so a job can be ordered relative to other units (e.g., "don't run this backup job until the database service is confirmed up") in a way cron has no concept of. Timers log through `journalctl` automatically with full output capture, timestamps, and exit status, whereas cron traditionally requires you to redirect output yourself and often loses failures silently unless you've built your own logging/alerting around it. `Persistent=true` (as in the lab) automatically catches up missed runs after downtime, which cron doesn't do out of the box. And `systemctl list-timers` gives immediate visibility into next/last run times for every timer on the system, versus manually parsing multiple users' crontabs to build the same picture.

**3. A `systemctl mask`'d service shows as masked in `systemctl status`, but a colleague insists they're able to start it via a different method. What might they actually be doing, and is that a real gap in mask's protection?**

They might be starting the underlying binary/process directly (e.g., running the executable manually rather than through systemd at all) — masking only prevents systemd itself from being able to activate the unit; it does nothing to prevent someone from bypassing systemd entirely and running the program directly from the command line. This isn't a gap in mask's guarantee about systemd-mediated activation specifically — mask does exactly what it promises (no systemd-managed start, ever, via any systemd trigger) — but it's a good reminder that mask isn't a security boundary preventing all possible ways of running a piece of software, just a systemd-level guarantee.

**4. Explain socket activation — what problem does it actually solve, and give a scenario where it's genuinely useful versus just having the service start normally at boot.**

Socket activation lets systemd own the listening socket itself and only actually start the associated service the moment a connection arrives on that socket, rather than the service needing to be running continuously in advance. This is genuinely useful for infrequently-used services where you don't want to pay the resource cost of keeping them running 24/7 just in case — for example, an SSH-triggered administrative tool or an internal debugging service that's rarely used, where socket activation means it consumes zero resources until the one moment someone actually connects, at which point systemd transparently starts it just in time. It's also useful during boot ordering — a service can be socket-activated so that OTHER services depending on it can start immediately and connect successfully, with the actual service starting lazily on first connection rather than needing to have fully started first, decoupling startup ordering from actual readiness timing in some architectures.

**5. Walk me through your debugging steps if a `systemctl enable --now` on a new timer unit succeeds with no errors, but the associated service never actually runs at the scheduled time.**

First, `systemctl list-timers` to confirm the timer is actually active and check its reported "Next" run time — if that's wrong or missing, the `OnCalendar=` syntax itself might be malformed (systemd is often lenient about accepting syntax that doesn't parse the way you intended, rather than erroring loudly). Next, I'd check `systemctl status <name>.timer` specifically (not the `.service`) to confirm the timer unit itself is active and enabled, since it's possible the `.timer` unit was never actually started even if the `.service` file looks fine. If the timer looks correct and active, I'd check `journalctl -u <name>.timer` for any timer-specific errors, and separately verify system time itself is correct (`timedatectl status`) — a system with drifted or incorrect time can cause a timer's calculated next-run to be wildly off from what you'd expect, which is an easy thing to overlook when you're staring only at the timer unit's own configuration.

---

## Topic 4: journalctl Deep Dive — Filtering, Persistent Logging, Cross-Boot Correlation

### Quick Review
- Journal storage may be **volatile** (RAM-only, lost on reboot) unless `/var/log/journal` exists and persistent logging is configured.
- Filter by unit, time range, priority, boot ID, and even structured fields (`_PID=`, `_UID=`).
- `journalctl --list-boots` + `journalctl -b -1` are essential for investigating a crash/unexpected reboot.
- `journalctl -o json-pretty` gives structured output for scripting/log shipping.

### Quick Learning

journalctl's real power for interview-level troubleshooting is **correlation** — connecting a specific failure to the exact boot, time window, and set of surrounding kernel/service events, rather than treating logs as an undifferentiated stream. The specific skill being tested: given "the server crashed and came back up, what happened," can you reconstruct the story from the journal rather than just running `journalctl` with no arguments and scrolling.

**Correlating an unexpected reboot — the actual workflow:**
```
  journalctl --list-boots
       │
       ▼
  Find the boot ID for the crash (the one before the CURRENT boot)
       │
       ▼
  journalctl -b -1 -p err          ──▶  What errors occurred right
                                        before the crash?
       │
       ▼
  journalctl -b -1 -k | tail -50   ──▶  Last KERNEL messages before
                                        the boot ended — OOM kill?
                                        Panic? Clean shutdown signal?
       │
       ▼
  journalctl -b -1 --since "5 min before last timestamp seen above"
       │
       ▼
  Cross-reference with the SPECIFIC service that was reportedly
  affected — journalctl -u <service> -b -1 — to see its own
  last logged activity relative to the kernel-level event
```

### Implementation (Learn by Applying)

**Scenario:** Set up persistent logging (often not configured by default on minimal installs), then practice the full crash-correlation workflow using a simulated OOM-style event.

```bash
# Check current journal storage mode
journalctl --disk-usage
cat /etc/systemd/journald.conf | grep -i storage

# Enable persistent logging if it's currently volatile
mkdir -p /var/log/journal
systemd-tmpfiles --create --prefix /var/log/journal
systemctl restart systemd-journald
```

Simulate a scenario worth correlating — force a memory-pressure event deliberately (lab-safe, adjust size to your VM's actual RAM):
```bash
# CAUTION: this deliberately consumes memory - only run on a lab VM, watch it closely
stress-ng --vm 2 --vm-bytes 90% --timeout 30s 2>/dev/null || \
  python3 -c "
import time
data = []
try:
    for i in range(1000):
        data.append(bytearray(50*1024*1024))
        time.sleep(0.1)
except MemoryError:
    pass
"
```

Now practice the actual correlation workflow:
```bash
journalctl --list-boots
journalctl -k | grep -iE "oom|out of memory|killed process"
journalctl -p err --since "10 minutes ago"

journalctl -g "killed process"           # -g does a regex grep directly within the journal
journalctl _PID=$$ 2>/dev/null           # Filter by an exact PID if you've identified a specific process of interest
journalctl -o json-pretty -u sshd --since "5 minutes ago" | head -30
```

### Interview Questions — with Answers

**1. A server rebooted unexpectedly overnight with nobody watching. Walk me through your exact journalctl-based investigation, step by step.**

First, `journalctl --list-boots` to enumerate every recorded boot session and identify the boot ID for the session that ended in the crash (typically `-1` relative to the current boot, assuming no reboots since). Then `journalctl -b -1 -p err` to surface any error-priority messages logged during that final session, giving an initial signal of what was going wrong before it ended. Critically, `journalctl -b -1 -k | tail -50` to check the very last kernel messages of that boot — this distinguishes between an OOM kill (`journalctl -k | grep -i oom`), a kernel panic, a hardware-related error, or simply a clean, intentional shutdown/reboot signal, each of which points investigation in a completely different direction. Finally, I'd check whatever service was reportedly affected specifically via `journalctl -u <service> -b -1` to see its own last activity and cross-reference the timing against the kernel-level finding.

**2. Why might a server's journal logs simply not exist after a reboot, even though you know the service was definitely logging normally before the crash?**

By default on some minimal RHEL installs, the journal is configured for **volatile** storage — kept only in RAM (`/run/log/journal`) — specifically so that logging doesn't consume persistent disk space unless explicitly configured otherwise. A RAM-only journal is entirely lost the instant the system reboots, by design, which means exactly the crash you most need to investigate is the one whose logs are gone. The fix is ensuring `/var/log/journal` exists and running `systemd-tmpfiles --create` plus restarting `systemd-journald`, which switches to persistent, disk-backed storage that survives reboots — this is a checklist item worth confirming BEFORE an incident, not after.

**3. What's the difference between `journalctl -p err` and `journalctl -p err..alert`, and when would the range syntax actually matter?**

`-p err` shows only messages logged at exactly the `err` priority level and nothing else. `-p err..alert` shows a range spanning from `err` up through the more severe levels (`crit`, `alert`, and implicitly `emerg` depending on exact range direction/interpretation) — priority levels in syslog/journald are ordered by severity, and the range syntax lets you capture "this severity or worse" in one filter rather than running separate queries per level. The range matters when you specifically want to see the highest-severity subset of messages without wading through lower-severity noise like `warning` or `notice`, but also don't want to miss something logged at `crit` or `alert` just because you only asked for `err` exactly.

**4. You need to feed journal logs into an external log-shipping/monitoring pipeline. What journalctl option makes that integration significantly easier, and why?**

`-o json-pretty` (or `-o json` for compact, single-line-per-entry output better suited to line-based log shippers) outputs each journal entry as structured JSON rather than journalctl's default human-readable text format — this matters because external tools (log aggregators, SIEM pipelines, custom scripts) can parse structured fields reliably (timestamp, priority, unit, message, PID, and dozens of other metadata fields journald tracks) instead of having to regex-parse journalctl's text output, which is fragile and can break if the text format ever changes slightly across versions. It's the difference between building a brittle text-scraping integration and a reliable, forward-compatible structured one.

**5. Explain what `journalctl -g "pattern"` does differently from piping `journalctl` output into `grep`, and why the difference might actually matter in practice.**

`journalctl -g` performs the pattern match natively within journalctl itself, directly against the journal's indexed data, before formatting and outputting results — versus piping into `grep`, where journalctl first has to render every single log line as text output (potentially a huge volume for a wide time range) and THEN grep filters that already-fully-rendered text stream. In practice, for very large journals or wide time ranges, `-g` can be meaningfully more efficient since it avoids fully formatting entries that will just get discarded by the external grep anyway — and it also respects journalctl's other filters (`-u`, `-p`, `--since`) in combination naturally within the same query, rather than needing careful pipe ordering to combine journalctl's own filters with an external grep correctly.

---

## Topic 5: Troubleshooting Boot Failures — Rescue/Emergency Targets and systemd-analyze

### Quick Review
- **`emergency.target`** — minimal environment, root shell only, almost nothing mounted; used when the system can't even reach basic functionality.
- **`rescue.target`** — single-user mode equivalent, most local filesystems mounted, more usable for actual repair work.
- A failing unit that's `Required=` by something on the critical boot path can hang or fail the ENTIRE boot.
- `systemctl --failed` after a successful-but-degraded boot shows exactly what didn't come up, without hunting through full logs.

### Quick Learning

The RHCSA-level version of this knowledge is "how do I reset a root password using rescue mode." The product-company-interview version is "a server is stuck at boot / boots into emergency mode, diagnose and fix it live" — a fundamentally different, more advanced skill, because you're reasoning about a genuinely broken dependency graph in real time rather than following a memorized recovery procedure for one specific known scenario.

**Why the system drops to emergency vs. rescue vs. just boots degraded:**
```
  Normal boot path:  sysinit.target ──▶ basic.target ──▶ multi-user.target
                                                                  │
                                                                  ▼
                                                         (system fully up)

  A CRITICAL unit fails (e.g., a filesystem in fstab without
  "nofail" can't mount) ──▶ boot HALTS, drops to:

     emergency.target        (if the failure is early/severe —
                                minimal mounts, root shell only,
                                you often can't even write logs
                                to a normal path yet)

  A less catastrophic but still blocking failure, or manual
  invocation for planned maintenance ──▶

     rescue.target            (most filesystems ARE mounted,
                                more services available, much
                                more livable for actual repair work)

  A NON-critical unit fails (Wants=, not Requires=, and not on
  the critical path) ──▶ boot CONTINUES to multi-user.target
  anyway, just missing that one thing:

     systemctl --failed       (shows you exactly what silently
                                didn't come up, in an otherwise
                                "successful" boot)
```

### Implementation (Learn by Applying)

**Scenario:** Deliberately break a critical fstab entry (no `nofail`) to force an emergency-mode boot, diagnose it live using only the tools available in that minimal environment, fix it, and reboot cleanly — then separately practice the "boots fine but one thing is missing" scenario, which is subtler and more common in real incidents.

```bash
# Deliberately add a broken, CRITICAL (no nofail) fstab entry pointing at a device that won't exist next boot
echo "/dev/nonexistent_device /brokenmount xfs defaults 0 2" >> /etc/fstab
reboot
```

At the emergency shell that follows:
```bash
# The environment is minimal - / is often mounted read-only, journal may not be writable normally
journalctl -xb                       # -x adds explanatory help text to failure messages, useful in this exact scenario

mount -o remount,rw /                # Make root writable so you can actually edit fstab
vi /etc/fstab                        # Remove or fix the broken entry
mount -a                              # Validate the fix works before rebooting
exit                                  # Or `systemctl default` to attempt continuing the boot without a full reboot
```

Separately, practice the subtler "boots fine but something's missing" case:
```bash
# Create a unit that will deliberately fail, but is only Wanted (not Required) by multi-user.target
cat > /etc/systemd/system/willfail.service << 'EOF'
[Unit]
Description=Deliberately fails for lab purposes

[Service]
Type=oneshot
ExecStart=/bin/false

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable willfail.service
reboot
```
After this reboot, the system comes up completely normally — SSH works, everything looks fine — but:
```bash
systemctl --failed              # Reveals willfail.service sitting there, silently failed, the whole time
systemctl status willfail.service
journalctl -u willfail.service -b
```

### Interview Questions — with Answers

**1. What's the actual difference between `emergency.target` and `rescue.target`, and how would you know, just from what's available to you at the shell, which one you've landed in?**

`emergency.target` is the more minimal of the two — it's invoked earliest/for more severe failures, with very little mounted (often just the root filesystem, and sometimes read-only) and almost no services running beyond a basic shell; you may not even have full logging or normal command availability. `rescue.target` is more hospitable — it mounts local filesystems more completely and brings up more of the basic system, closer to old single-user mode, making it more usable for actual repair work. Practically, I'd know which one I'm in by checking what's mounted (`mount` shows far less in emergency mode), whether `/etc/fstab`-defined filesystems beyond root are available, and the prompt/messaging systemd itself displays when dropping you into either target, which explicitly names which target you've landed in.

**2. A critical fstab entry (no `nofail`) points at a disk that's failed. Walk me through exactly what you'd do at the emergency shell to get the system back to a normal boot, including any read-only filesystem gotchas.**

First, `journalctl -xb` to confirm the actual failure and get systemd's own explanatory context for why boot halted — this confirms my hypothesis rather than assuming. Since the root filesystem may be mounted read-only in this state, my next step is `mount -o remount,rw /` specifically, BEFORE attempting to edit anything — trying to edit `/etc/fstab` on a read-only root without this step fails with a permissions-looking error that's actually a mount-state issue, a common point of confusion. Then I'd edit `/etc/fstab` to remove or fix the broken entry (or add `nofail` if the disk is expected to sometimes be legitimately absent), run `mount -a` to validate the fix works before committing to a full reboot, and either `reboot` or attempt `systemctl default` to continue booting without a full restart.

**3. Why does adding `nofail` to a non-critical fstab entry matter operationally, beyond just "it prevents emergency mode"? Give a concrete example of the downstream impact if it's missing.**

Without `nofail`, a missing/failed mount for a genuinely non-critical filesystem (say, an optional data drive that isn't needed for the OS or core services to function) can take the ENTIRE system down into emergency mode over something that shouldn't have been boot-blocking at all — meaning a single non-essential disk failing turns into a full outage requiring manual console/IPMI intervention to even get SSH access back, rather than the system simply booting normally with that one mount point unavailable (and logging/alerting on that fact separately). Concrete example: a backup/archive drive that occasionally has connectivity hiccups on a VM/cloud environment shouldn't be able to take down a production web server's ability to boot and serve its actual primary workload — `nofail` is exactly the mechanism that decouples "this specific mount is unavailable" from "the entire boot fails."

**4. A server boots completely normally — SSH works, the application appears fine — but you later discover a service had actually failed to start and nobody noticed for days. How did this happen, and how do you prevent it going forward?**

This happens because the failed service was only `Wanted=`, not `Required=`, by the target it was hooked into — a soft dependency means its failure doesn't block or even visibly disrupt the rest of the boot sequence, so the system reaches a fully "successful," normal-looking boot state despite this one component silently not being there. Going forward, I'd build active monitoring around `systemctl --failed` (or the equivalent status check via a monitoring agent) run periodically, or triggered right after every boot specifically, rather than relying on someone noticing an application-level symptom eventually — `systemctl --failed` is a direct, authoritative source of "what didn't come up" that shouldn't require guesswork or waiting for a downstream symptom to surface.

**5. Explain why `systemctl --failed` is a better first command to run after "something seems off but the server is technically up" than scrolling through general system logs.**

`systemctl --failed` gives you a direct, pre-filtered, authoritative list of exactly which units systemd itself considers to be in a failed state right now — it's systemd's own summary judgment, not something you have to infer by reading through potentially thousands of unrelated log lines and mentally reconstructing which services succeeded versus failed. Scrolling general logs requires knowing roughly what you're looking for and when it happened; `--failed` requires neither — it's the fastest way to get a definitive, complete answer to "is anything actually broken right now" before you invest time in log archaeology for a specific service you haven't even identified as the problem yet.

---

## Topic 6 (Bonus — Advanced): Resource Control via systemd — cgroups, Slices, and Per-Service Limits

### Quick Review
- Modern systemd (RHEL 8/9) manages **cgroups v2** directly — every service runs inside its own cgroup automatically.
- `CPUQuota=`, `MemoryMax=`, `IOWeight=` in a unit file enforce hard resource limits per service, no separate cgroup tooling required.
- `systemd-cgtop` shows live resource consumption per cgroup/service, similar in spirit to `top` but organized by systemd's own hierarchy.
- A runaway service hitting `MemoryMax=` gets OOM-killed by the kernel specifically for exceeding ITS limit — not a general system-wide OOM event.

### Quick Learning

This is the topic that most directly bridges "systemd knowledge" into "capacity/performance work," and it's exactly the kind of question that separates someone who's only used systemd for start/stop/enable from someone who understands it as the actual resource-management layer it's become in modern RHEL. Every service already runs inside a cgroup whether you've configured anything or not — systemd creates the hierarchy automatically — the only question is whether you've set meaningful limits on it, or left it able to consume unlimited resources and potentially starve everything else on the box.

**One noisy service, contained versus uncontained:**
```
  WITHOUT resource limits                  WITH MemoryMax=/CPUQuota= set
  ────────────────────────                 ─────────────────────────────
  myapp.service                            myapp.service
  ├─ memory leak, growing unbounded         ├─ MemoryMax=2G
  │                                          │
  ▼                                          ▼
  Eventually consumes ALL system            Hits 2G, kernel OOM-kills
  memory ⇒ kernel OOM-killer picks           JUST this service's cgroup
  SOME process to kill — possibly            ⇒ systemd sees it fail,
  a completely unrelated, healthy            applies Restart= policy
  service, because the OOM-killer            ⇒ OTHER services on the
  doesn't know which process is              box were never touched,
  actually "at fault" system-wide            never even noticed anything
```

### Implementation (Learn by Applying)

**Scenario:** A service has a known memory leak that the vendor hasn't fixed yet. Rather than letting it eventually take down the whole server via an unpredictable system-wide OOM event, contain it with a hard resource ceiling so its failure is isolated and automatically recovered.

```bash
systemctl edit myapp.service
```
Add to the drop-in:
```ini
[Service]
MemoryMax=1500M
MemoryHigh=1200M
CPUQuota=150%
IOWeight=50
```
```bash
systemctl daemon-reload
systemctl restart myapp.service

# Watch it live
systemd-cgtop

# Confirm the limits actually took effect at the cgroup level
systemctl show myapp.service -p MemoryMax -p CPUQuotaPerSecUSec
cat /sys/fs/cgroup/system.slice/myapp.service/memory.max
```

Simulate hitting the limit and observe the contained failure:
```bash
systemctl status myapp.service --no-pager
# (After the leak grows past MemoryMax, expect to see it OOM-killed and restarted per its Restart= policy)
journalctl -u myapp.service | grep -i "killed\|oom"
```
`MemoryHigh=` (set below `MemoryMax=`) is worth understanding as distinct from `MemoryMax=`: `MemoryHigh` is a *soft* throttling point — the kernel starts aggressively reclaiming memory and slowing the cgroup down as it approaches this value, giving the service a chance to recover gracefully — while `MemoryMax` is the hard ceiling where the kernel simply kills something in the cgroup outright. Using both together gives you graduated pressure instead of a single hard cliff.

### Interview Questions — with Answers

**1. Explain why, even if you've never explicitly configured any resource limits, every systemd service is still running inside a cgroup. What does that mean practically?**

Modern systemd (on cgroups v2, the default in RHEL 8/9) automatically creates a cgroup for every unit it manages as a fundamental part of process tracking and lifecycle management — this isn't an opt-in feature, it's baked into how systemd supervises services at all, used for things like correctly identifying and cleaning up all of a service's child processes on stop, regardless of whether any resource LIMITS have been explicitly set. Practically, this means the infrastructure for resource control is already there and already organizing every service into its own accounting boundary — configuring `MemoryMax=`/`CPUQuota=` is just attaching enforcement to a structure that already exists, not building something new from scratch.

**2. What's the practical difference between `MemoryMax=` and `MemoryHigh=`, and why would you configure both together rather than just setting a single hard limit?**

`MemoryMax=` is a hard ceiling — exceeding it results in the kernel OOM-killing something within that cgroup, a hard, disruptive stop. `MemoryHigh=` is a softer throttling threshold set below `MemoryMax=` — crossing it causes the kernel to apply memory pressure and aggressively reclaim reclaimable memory from the cgroup, effectively slowing it down and giving it a chance to reduce its own footprint (via garbage collection, cache eviction, etc.) before ever reaching the hard `MemoryMax=` kill point. Configuring both together gives you a graduated response — mild throttling first as an early warning the service itself might recover from, with the hard kill reserved as a genuine last resort — rather than a single cliff-edge where the only outcome is an abrupt kill.

**2b. If a service hits its configured `MemoryMax=` and gets OOM-killed, does that affect other unrelated services on the same box? Why or why not?**

No — because the kill happens within the scope of that service's own cgroup specifically, the kernel's cgroup-aware OOM killer targets processes within the offending cgroup, leaving other cgroups (other services) on the system completely untouched. This is precisely the operational benefit over a traditional system-wide OOM event: without per-service limits, a single leaking service consuming all system memory triggers the kernel's global OOM-killer, which can end up killing an entirely unrelated, healthy process simply because the kernel's heuristics picked it as the "best" candidate to free memory — with cgroup limits in place, the failure is contained precisely to the service that actually caused it.

**3. How would you determine an appropriate `MemoryMax=` value for a service you don't have historical baseline data for yet?**

I'd start conservatively high (or without a hard limit initially, using only `MemoryHigh=` as an early-warning soft throttle) while actively monitoring actual consumption via `systemd-cgtop` or the equivalent metric from a monitoring agent over a representative period of real production load, including peak periods. Once I have that baseline, I'd set `MemoryMax=` with meaningful headroom above observed peak usage — enough to avoid false-positive kills during legitimate load spikes, but still low enough to actually contain a genuine leak or runaway condition well before it threatens total system memory. I'd treat the initial value as provisional and revisit it once more real-world data accumulates, rather than picking an arbitrary number upfront with no baseline at all.

**4. What's the difference between `systemd-cgtop` and plain `top` — when would you specifically reach for `systemd-cgtop` instead?**

Plain `top` organizes and displays resource consumption per individual process, which can be hard to interpret for a service that spawns multiple child processes (a web server with worker processes, for example) — you'd have to mentally sum multiple `top` rows to understand that service's TOTAL footprint. `systemd-cgtop` organizes consumption by cgroup — meaning by systemd unit/service — automatically aggregating all of a service's child processes into one figure, which directly answers "how much is THIS SERVICE, in total, actually consuming" without manual aggregation. I'd reach for `systemd-cgtop` specifically when investigating a suspected resource issue tied to a particular service (especially a multi-process one) rather than trying to identify which individual PID among many is the actual culprit.

**5. `CPUQuota=150%` is set on a service. What does that percentage actually mean on a multi-core system, and what mistake do people commonly make interpreting it?**

`CPUQuota=` percentages are expressed relative to a SINGLE CPU core's worth of time, not the whole system — `150%` means the service is allowed to consume the equivalent of 1.5 full CPU cores' worth of processing time, which on an 8-core system is a meaningful cap well below the system's total capacity, even though "150%" sounds like a large number in isolation. The common mistake: someone sees "150%" and assumes it means "150% of the whole system," either wildly under-provisioning a service that actually needs more (if they thought 150% was generous when it's actually quite modest on a many-core box) or misunderstanding why a service still hits its quota and gets throttled despite the system overall showing plenty of idle CPU — the quota is a per-service ceiling on core-equivalents, not a relative share of total system capacity.

---

**End of Day 3.** You should now be able to reason about the systemd dependency graph precisely (the `After=` vs `Requires=` distinction, and why it causes real bugs), author and safely customize unit files via drop-ins, correctly choose between disable and mask, replace cron with timers where the added dependency-awareness and logging genuinely help, run the full journalctl crash-correlation workflow across boots, diagnose and recover from emergency/rescue-mode boot failures live, and — as this day's advanced capstone — use systemd's built-in cgroup integration to contain a misbehaving service's resource consumption before it becomes everyone else's problem.

Proceed to **Day 4 — Identity & Access (User Management)** next.
