# Day 10 — Performance & Package Management (Full Study Edition, with Answers)

**Estimated time: 5–6 hours**

This is the final core day, and it's deliberately built as a capstone: "the server is slow, diagnose it" is one of the most common product-company interview scenarios precisely because it draws on everything from the previous nine days at once — storage (Day 1), networking (Day 2), systemd/cgroups (Day 3), and the diagnostic commands Chapter 0 introduced but never fully connected into one workflow. Today covers `tuned-adm` profiles, a proper CPU/memory bottleneck escalation, a proper disk/network bottleneck escalation, package management depth beyond basic `dnf install`, and — as the true capstone — a single, unified "server is slow" decision tree that ties all ten days together into one coherent diagnostic instinct.

---

## Topic 1: tuned-adm — Performance Profiles, Not Magic

### Quick Review
- `tuned` applies a coordinated SET of kernel/system tunings (sysctl values, I/O scheduler, CPU governor, and more) as a named, switchable PROFILE, rather than requiring manual, individual tuning of each underlying parameter.
- `tuned-adm recommend` suggests a profile based on detected hardware/virtualization environment — a reasonable starting point, not a guaranteed-correct final answer for every workload.
- Common profiles: `throughput-performance` (general server workloads prioritizing throughput), `latency-performance` (minimizing latency at some throughput cost), `virtual-guest`/`virtual-host` (VM-specific tuning).
- A profile is a coordinated STARTING POINT — real production tuning often still requires workload-specific adjustments on top of whatever profile is active, tying directly back to Day 9's "tune based on evidence, not a checklist" lesson.

### Quick Learning

`tuned-adm` exists precisely because manually tuning dozens of individual kernel parameters coherently, for a specific workload TYPE, is genuinely complex and easy to get partially wrong (adjusting one parameter without adjusting related ones that should move together) — a profile bundles a whole set of parameters known to work well TOGETHER for a given workload category. The trap candidates fall into: treating `tuned-adm profile X` as if it's a complete, sufficient answer to "how do you tune this server" — the correct, deeper answer is that profiles are a sensible, coordinated STARTING baseline, and genuine production tuning still requires the evidence-based, symptom-driven approach from Day 9's sysctl topic on top of whatever profile provides.

**What a profile actually bundles together, versus tuning parameters individually:**
```
  Manually tuning individual parameters:        tuned-adm profile throughput-performance:
  ──────────────────────────────────────         ───────────────────────────────────────
  sysctl vm.dirty_ratio=...                      ONE command applies a coordinated,
  sysctl vm.swappiness=...                        pre-designed BUNDLE of all of these
  echo performance > cpu governor                 together — values chosen to work
  echo deadline > I/O scheduler                   COHERENTLY as a set, not just
  ... (dozens more, each requiring its OWN        individually reasonable in isolation
  correct value AND correct interaction
  with every other related setting)               tuned-adm profile virtual-guest
                                                    switches to an ENTIRELY DIFFERENT
                                                    coordinated bundle, appropriate
                                                    for a VM guest's different
                                                    performance characteristics —
                                                    switching workload TYPE means
                                                    switching the whole bundle, not
                                                    hand-adjusting individual values
```

### Implementation (Learn by Applying)

**Scenario:** A database server is currently running the default profile but showing signs of I/O-related latency issues under load. Investigate the current profile, understand what changing it would actually affect, and make an evidence-based decision about switching.

```bash
tuned-adm active
tuned-adm list
tuned-adm recommend                    # What tuned itself suggests, based on detected environment

# See exactly what a candidate profile actually CHANGES, before blindly switching to it
tuned-adm profile_info throughput-performance 2>/dev/null || \
  cat /usr/lib/tuned/throughput-performance/tuned.conf
```

Gather the actual evidence BEFORE switching (per Day 9's evidence-based tuning principle):
```bash
iostat -xz 2 5                         # Check current I/O latency/utilization pattern
vmstat 2 5                             # Check current CPU/memory pressure pattern
```

Switch profiles based on the gathered evidence, and confirm the change actually took effect at the underlying parameter level:
```bash
tuned-adm profile throughput-performance
tuned-adm active

cat /sys/block/sda/queue/scheduler 2>/dev/null      # Confirm the I/O scheduler actually changed as part of the profile switch
sysctl vm.dirty_ratio                                  # Confirm a specific bundled sysctl value changed too
```

### Interview Questions — with Answers

**1. Why does `tuned-adm` exist as a profile-based system rather than simply documenting a list of recommended sysctl values for administrators to apply manually?**

Coherent performance tuning for a specific workload type typically requires MULTIPLE related parameters to be adjusted TOGETHER in a coordinated way — the I/O scheduler, CPU governor, various sysctl values, and more all interact with each other, and adjusting one in isolation without considering how it relates to the others can produce a partially-tuned, internally inconsistent configuration that doesn't actually achieve the intended coherent performance characteristic. `tuned-adm`'s profile system exists specifically to bundle a whole, coordinated set of parameters known to work well TOGETHER as a named, switchable unit — reducing both the complexity of correctly tuning many interacting parameters and the risk of an administrator manually adjusting some but not all of the genuinely-related settings a specific workload actually needs.

**2. `tuned-adm recommend` suggests a profile based on the detected environment. Why shouldn't you treat this recommendation as automatically correct for every workload running on that server?**

`tuned-adm recommend`'s suggestion is based on detectable environmental characteristics (is this a VM or bare metal, general hardware class, and similar broad signals) — it has no actual knowledge of what SPECIFIC WORKLOAD is running on this particular server, or what that workload's actual performance characteristics and bottlenecks genuinely are. A server that's technically bare-metal hardware but running a latency-sensitive application would get a generic bare-metal recommendation that might prioritize throughput over latency in a way that's actually wrong for that specific workload's real needs — the recommendation is a reasonable, environment-aware STARTING guess, not a substitute for actually understanding your specific application's performance requirements and validating the chosen profile against real, observed behavior under real load, exactly the evidence-based principle from Day 9.

**3. A team switches to `tuned-adm profile throughput-performance` expecting improved database performance, but observes no meaningful change. What would you investigate, given what you know about how profiles actually work?**

First, I'd confirm the profile switch actually took effect at the underlying parameter level — checking specific bundled settings the profile is supposed to change (I/O scheduler, relevant sysctls) directly, rather than just trusting `tuned-adm active`'s reported profile name, since it's possible something else (a conflicting manual tuning, a different mechanism overriding the same parameters) is preventing the profile's settings from actually taking hold. If the profile genuinely did apply correctly, I'd reconsider whether the ACTUAL bottleneck this database workload is experiencing is even something this profile's bundled tunings address at all — `throughput-performance` optimizes for a certain kind of I/O/CPU pattern, but if the real bottleneck is, say, network latency, application-level query inefficiency, or a resource constraint the profile has no influence over whatsoever, switching profiles wouldn't be expected to help regardless of how correctly it applied — this connects back to needing actual diagnostic evidence (this day's later topics) BEFORE assuming a profile change is even the right category of fix.

**4. What's the practical difference between `virtual-guest` and `throughput-performance` profiles conceptually, and why would running the WRONG one for a given environment potentially cause problems rather than just being a missed optimization?**

`virtual-guest` is tuned specifically for a server running AS a virtual machine, accounting for the fact that certain tuning assumptions appropriate for bare-metal hardware (direct hardware access assumptions, certain timing-sensitive optimizations) don't hold the same way inside a virtualized environment, where the hypervisor is itself managing underlying physical resource scheduling. `throughput-performance` assumes more direct control over and knowledge of the actual underlying hardware. Running `throughput-performance` on an actual VM (rather than the VM-aware `virtual-guest` profile) could apply tunings based on incorrect assumptions about direct hardware behavior that don't actually hold true inside a VM — potentially not just failing to help, but actively working against the hypervisor's OWN scheduling/resource management in ways that could genuinely degrade performance rather than simply representing an unoptimized-but-neutral missed opportunity.

**5. How would you validate that a `tuned-adm` profile change genuinely improved performance for a specific workload, rather than just trusting that switching profiles inherently helped?**

I'd establish a proper before/after comparison using the same evidence-gathering tools from this topic's implementation (`iostat`, `vmstat`, and the broader diagnostic toolkit from this day's later topics) under comparable, representative load conditions — capturing baseline metrics BEFORE the profile switch, applying the change, then capturing the same metrics again under similarly representative load AFTER, and comparing them directly rather than relying on a subjective sense that things "feel faster." This is the same evidence-based validation discipline from Day 9's sysctl topic, applied here specifically — a `tuned-adm` profile switch is still fundamentally a tuning change, and deserves the same rigor in confirming it actually achieved its intended effect for THIS specific workload, rather than assuming a named, seemingly-appropriate profile automatically delivers the improvement its name suggests.

---

## Topic 2: CPU & Memory Bottleneck Diagnosis — The Escalation Order

### Quick Review
- `top`/`htop` for an immediate, at-a-glance signal; `vmstat` to confirm whether it's genuinely CPU-bound, I/O-wait-bound, or swap-thrashing; `mpstat -P ALL` to catch a single hot core hidden by an aggregate view; `pidstat` to narrow down to the SPECIFIC process responsible.
- A high load average alone doesn't tell you WHETHER the bottleneck is CPU, I/O-wait, or something else — `vmstat`'s `r`/`b`/`wa` columns are what actually disambiguate.
- Swap activity (`si`/`so` in `vmstat`, non-zero and sustained) is a genuine red flag distinct from merely "swap is being used" (some baseline swap USAGE with zero ACTIVITY is often fine).
- `mpstat -P ALL` matters specifically because a SINGLE-THREADED bottleneck on one hot core can be completely invisible in an aggregate, whole-system CPU percentage.

### Quick Learning

The real skill here isn't knowing individual commands — Chapter 0 already covered their syntax — it's knowing the correct ESCALATION ORDER and what each step is specifically meant to DISAMBIGUATE, exactly the same "layer by layer, ruling out categories" discipline from Day 2's connectivity troubleshooting. A high load average by itself is genuinely ambiguous — it could mean genuine CPU contention, or it could mean many processes stuck waiting on slow I/O (which also counts toward load average on Linux) — and jumping straight to "the CPU is overloaded" without confirming via `vmstat`'s specific columns is exactly the kind of imprecise, symptom-matching answer that doesn't hold up under a real interviewer's follow-up questions.

**The CPU/memory escalation order, and what each step specifically rules in or out:**
```
  top / htop                          "Something's off" — quick, broad signal,
       │                              but doesn't yet tell you WHAT category
       ▼
  vmstat 2 5                          Disambiguates the CATEGORY:
       │                                r (run queue) high, wa low  → genuinely CPU-bound
       │                                b (blocked) high, wa high   → I/O-wait-bound, NOT
       │                                                                a CPU problem at all,
       │                                                                despite high load average
       │                                si/so NON-ZERO, sustained   → genuine memory pressure,
       │                                                                swap-thrashing
       ▼
  mpstat -P ALL 2                     If genuinely CPU-bound per vmstat: is it
       │                              spread evenly across cores, or is ONE
       │                              core pegged while others sit idle?
       │                              (a single-threaded bottleneck hides in
       │                               the AGGREGATE view top/vmstat show)
       ▼
  pidstat -u 2  /  pidstat -d 2       WHICH SPECIFIC PROCESS is responsible —
                                       narrows from "the system" to "this PID,"
                                       the actionable, specific answer
```

### Implementation (Learn by Applying)

**Scenario:** A server reports high load average, and the on-call engineer's first instinct is "we need more CPU cores" — work through the correct escalation to either confirm or REFUTE that hypothesis with actual evidence, deliberately including a case where the "obvious" answer is wrong.

Simulate an I/O-wait-heavy load (high load average that is NOT actually a CPU problem):
```bash
for i in 1 2 3 4; do
  dd if=/dev/zero of=/tmp/iotest_$i bs=1M count=2000 oflag=direct &
done
wait_pids=$(jobs -p)

uptime                                  # High load average observed
top -bn1 | head -15                     # CPU% might look surprisingly LOW despite the high load average — the key clue
```

Disambiguate correctly with vmstat:
```bash
vmstat 2 5
# Look specifically at: r (should be modest) vs b (likely HIGH — processes blocked on I/O)
# and wa (I/O wait percentage — likely HIGH), NOT genuine CPU contention (us/sy columns modest)
```
This confirms the load average is being driven by I/O-WAIT, not CPU contention — "add more CPU cores" would do NOTHING to fix this, since the bottleneck is disk throughput/latency, not CPU capacity.

Clean up and now simulate a GENUINE single-threaded CPU bottleneck, to see the contrasting signature:
```bash
rm -f /tmp/iotest_*

python3 -c "
import time
end = time.time() + 20
while time.time() < end:
    x = 2**1000000  # Deliberately CPU-intensive, single-threaded
" &

vmstat 2 5              # NOW: r is elevated, wa is low — genuinely CPU-bound this time
mpstat -P ALL 2 3        # Confirm: ONE core pegged near 100%, others near idle — a single-threaded bottleneck,
                          # invisible in top's aggregate percentage if the system has many cores
pidstat -u 2 3            # Identify the SPECIFIC process/PID responsible
```

### Interview Questions — with Answers

**1. A server shows a high load average, and someone concludes "we need more CPU capacity" based on that number alone. Why is that conclusion potentially wrong, and what would you check before agreeing or disagreeing?**

On Linux, load average includes processes in an UNINTERRUPTIBLE SLEEP state (commonly, processes blocked waiting on I/O), not just processes genuinely competing for CPU time — a high load average can be driven entirely by many processes stuck waiting on slow disk or network I/O, with the CPU itself sitting mostly IDLE the whole time, in which case adding more CPU cores would do absolutely nothing to help, since CPU was never the actual constraint. Before agreeing or disagreeing with the "need more CPU" conclusion, I'd check `vmstat`'s `r` (run queue, genuinely CPU-contending processes) versus `b` (blocked, typically I/O-waiting processes) columns specifically, alongside the `wa` (I/O wait percentage) column — a high `b`/`wa` with a modest `r` conclusively demonstrates the load average is I/O-driven, not CPU-driven, directly refuting the "add more CPU" conclusion with concrete evidence rather than continuing to argue based on the ambiguous load-average number alone.

**2. Explain why `mpstat -P ALL` can reveal a bottleneck that `top`'s aggregate CPU percentage completely hides, using a concrete example with numbers.**

`top`'s default aggregate CPU percentage AVERAGES utilization across every core — on an 8-core system, if ONE core is pegged at 100% (a genuine, severe bottleneck for whatever single-threaded process is running on it) while the other 7 cores sit completely idle at 0%, the AGGREGATE percentage `top` reports is only (100 + 0×7) / 8 = 12.5% — a number that looks like the system has abundant CPU headroom, completely masking the fact that ONE specific core, and whatever process is bound to it, is maximally saturated and likely the actual bottleneck for that specific workload. `mpstat -P ALL` breaks utilization down PER-CORE specifically, immediately revealing this exact pattern (one core at ~100%, others near 0%) that the aggregate view averages away into an innocuous-looking overall number.

**3. What's the meaningful difference between "some swap is currently being used" (a non-zero value in `free -h`'s swap line) and "the system is actively swap-thrashing" — why does conflating these two lead to incorrect troubleshooting?**

Some baseline swap USAGE (a non-zero amount of data currently sitting in swap) can be completely benign and even expected — the kernel may have proactively swapped out genuinely inactive, rarely-touched memory pages to free up RAM for more actively-used data, which is efficient, sensible memory management, not a problem. Active swap-THRASHING specifically means the kernel is CONTINUOUSLY moving pages between RAM and swap under current memory pressure — visible as sustained, non-zero `si`/`so` (swap in/out) ACTIVITY in `vmstat`, not just a static non-zero total swap USAGE figure — and this active churning is genuinely expensive and a real performance problem, since swap (disk-based) is vastly slower than RAM, and actively-needed pages are being repeatedly shuffled to and from it. Conflating "swap has some data in it" with "the system is thrashing" leads to false alarms (assuming a problem exists when swap usage is actually benign, static baseline usage) or, worse, missing a REAL thrashing problem if you only check total swap usage once rather than watching `si`/`so` activity over time to see if it's actively, continuously churning.

**4. Walk me through your exact diagnostic escalation, in order, if asked "the server feels slow, diagnose it" with no other information given.**

I'd start with `top`/`htop` for an immediate, broad signal — is anything obviously abnormal at a glance (a runaway process, obviously high load). Then `vmstat 2 5` specifically to disambiguate the CATEGORY of bottleneck — genuinely CPU-bound (`r` elevated, `wa` low) versus I/O-wait-bound (`b`/`wa` elevated) versus memory-pressure/swap-thrashing (sustained `si`/`so` activity) — since these require completely different further investigation paths. If CPU-bound per vmstat, `mpstat -P ALL` to check whether it's evenly distributed across cores or a single hot-core/single-threaded bottleneck hidden in aggregate views. Then `pidstat -u`/`pidstat -d` to narrow from "the system in general" down to the SPECIFIC process actually responsible. If the vmstat signature instead pointed toward I/O-wait, I'd pivot to Topic 3's disk-specific escalation (`iostat -xz`) instead of continuing down the CPU-specific path, since the category identified at the vmstat step determines which SPECIFIC further tools are actually relevant.

**5. A `pidstat -u` check identifies a specific process consuming high CPU, but that process is a legitimate, expected part of the application (not a bug or runaway process). What would your NEXT investigative step be, beyond just identifying which process it is?**

Identifying the specific responsible process answers "WHO is using the CPU," but not yet "WHY is this LEGITIMATE process needing this much CPU right now, and is that level of usage actually appropriate/expected for the current workload." I'd investigate the application's own context — is this consistent with genuinely increased legitimate load (more requests/users than usual, a normal and expected scaling pattern), or is it consuming MORE CPU than it should for the CURRENT actual workload level (suggesting an application-level inefficiency, an unoptimized code path recently deployed, or a configuration issue causing unnecessary work) — this typically means moving from system-level tools into application-specific profiling/logging/APM tooling, or correlating the CPU spike's timing against recent deployments/configuration changes/traffic patterns, since "which process" and "is this process's resource usage actually appropriate right now" are genuinely different questions requiring different follow-up investigation.

---

## Topic 3: Disk & Network I/O Bottleneck Diagnosis

### Quick Review
- `iostat -xz` is the primary disk I/O diagnostic tool — `%util` (device busy percentage), `await` (average I/O wait time), and `avgqu-sz` (queue depth) are the key columns.
- Sustained `%util` near 100% means the DEVICE itself is the bottleneck; a growing gap between `await` and the device's typical service time suggests QUEUEING, not raw device slowness.
- `sar -n DEV` and the broader `sysstat` toolkit provide network throughput history — critically, `sar`'s HISTORICAL retention (from Chapter 0) is what lets you investigate a bottleneck that occurred overnight, after the fact, when live tools alone couldn't have caught it.
- Disk and network bottleneck diagnosis should connect back to Day 1 (LVM/storage layering — WHICH physical layer is actually saturated) and Day 2 (network escalation order) rather than being treated as entirely separate diagnostic domains.

### Quick Learning

Disk and network I/O diagnosis is where this day's capstone framing becomes most concrete: a genuinely slow disk isn't just "check iostat" in isolation — it's connecting `iostat`'s device-level view back to Day 1's LVM layering (is the bottleneck at the PV/physical-disk level, or is it something happening at the LV/filesystem layer, like a thin pool approaching its physical limit) to actually understand WHERE in the storage stack the saturation is occurring, not just THAT it's occurring.

**iostat's key columns and what they actually mean, read together:**
```
  iostat -xz 2
  ─────────────
  Device   %util   await   avgqu-sz
  sda      98.5    45.2    12.3

  %util near 100%:     the device is BUSY nearly all the time —
                        this alone tells you the device is likely
                        the bottleneck, but not WHY

  await HIGH relative   if await is high AND avgqu-sz is high,
  to typical device      requests are QUEUEING UP faster than the
  service time:           device can drain them — genuine saturation,
                          not just one slow individual operation

  avgqu-sz consistently  confirms the QUEUE itself is backing up —
  above ~1-2:             a device that's fast enough for the
                           current load would keep this near zero
                           even under significant throughput

  Compare against Day 1's storage layering: is THIS specific
  device (a single PV) the bottleneck, or is the actual constraint
  somewhere else in the stack — a thin pool nearing its physical
  limit (Day 1, Topic 6), a specific LV striped in a way that's
  concentrating load unevenly, or genuinely just this one physical
  disk being under-provisioned for the current workload
```

### Implementation (Learn by Applying)

**Scenario:** An application reports slow database writes. Work through the disk I/O escalation to confirm whether the disk is genuinely the bottleneck, then correlate the finding back to Day 1's storage-layer knowledge to identify WHERE specifically in the stack the constraint lives.

```bash
# Generate representative write load against a test volume
dd if=/dev/zero of=/appdata/iotest bs=4k count=100000 oflag=direct &
DD_PID=$!

iostat -xz 2 5
```
Focus specifically on `%util`, `await`, and `avgqu-sz` for the device(s) backing `/appdata` — confirm whether they show the saturation signature described above.

```bash
wait $DD_PID
rm -f /appdata/iotest
```

Connect back to Day 1 — confirm WHICH layer of the storage stack the bottleneck actually corresponds to:
```bash
lvs -o +devices /dev/vg_app/lv_app     # Which PHYSICAL disk(s) actually back this LV?
pvs                                      # Is this specific PV showing signs of being individually saturated,
                                           # or is the LV striped/spread such that the bottleneck might be
                                           # concentrated on just ONE of several PVs it spans?
```

For network throughput specifically, and its historical-investigation advantage:
```bash
sar -n DEV 1 5                           # Live network throughput per interface, right now
sar -n DEV -f /var/log/sa/sa$(date -d yesterday +%d) 2>/dev/null | head -30   # HISTORICAL data —
                                            # exactly what lets you investigate an overnight bottleneck
                                            # after the fact, which top/iostat/vmstat alone never could
```

### Interview Questions — with Answers

**1. `iostat -xz` shows a device at 98% `%util`. Does this definitively mean the disk hardware itself is too slow and needs to be replaced/upgraded? What else could this actually indicate?**

Not necessarily — high `%util` means the device is BUSY nearly the entire time, but this could reflect the device genuinely being under-provisioned for legitimate current load (a real case for upgrading), OR it could reflect something ELSE generating excessive, potentially unnecessary I/O against a device that would otherwise be perfectly adequate — an application performing far more disk operations than actually necessary (inefficient query patterns, excessive logging, a misconfigured sync frequency), a thin-provisioned volume (Day 1) approaching genuine physical exhaustion and struggling under that pressure, or even competing I/O from an unrelated process/container sharing the same underlying physical device. High `%util` tells you WHERE to look (this device is saturated), not automatically WHY — correlating with `avgqu-sz`/`await`, checking what's actually GENERATING the I/O (`pidstat -d`), and understanding the full storage stack context (Day 1) are all necessary before concluding "replace the hardware" is the right fix versus "something's generating more I/O than it should."

**2. Explain the relationship between `await` and `avgqu-sz` in `iostat` output — why do you need to look at them TOGETHER rather than either one in isolation to correctly diagnose genuine disk saturation?**

`await` alone (average time an I/O request takes to complete) could be elevated for reasons unrelated to genuine device saturation — a single, unusually large or complex I/O operation might legitimately take longer without indicating a systemic bottleneck. `avgqu-sz` alone (average queue depth) tells you requests are backing up, but not necessarily how BADLY that's affecting actual completion times. Looking at them TOGETHER is what confirms genuine saturation: a HIGH `avgqu-sz` (many requests queued, waiting) combined with a correspondingly HIGH `await` (those queued requests are genuinely taking a long time to actually complete, not just sitting briefly before rapid processing) together confirm the device is genuinely unable to keep up with the current request rate — requests are piling up AND taking meaningfully longer as a direct result, which is the actual signature of saturation, distinct from either metric being elevated in isolation for some other, less systemic reason.

**3. Why is `sar`'s historical data retention specifically valuable for network/disk troubleshooting, in a way that `iostat`/`top`/`vmstat` alone fundamentally cannot replicate?**

`iostat`, `top`, and `vmstat` are all LIVE tools — they show you the system's current state at the moment you run them, with no memory of anything that happened before you started watching. If a performance incident occurred overnight or during a period nobody was actively monitoring in real time, these live tools have nothing to offer for investigating it after the fact — by the time you're looking, the system may be back to normal, and the live tools show a perfectly healthy current state that gives no indication anything was ever wrong. `sar`, via the `sysstat` package's scheduled data collection into `/var/log/sa/`, maintains a HISTORICAL RECORD of these same metrics over time — letting you go back and examine exactly what CPU/memory/disk/network metrics looked like at a SPECIFIC past time window, which is the only way to actually investigate an incident that already happened and ended before you started looking, a fundamentally different capability than any of the live-only tools provide.

**4. A disk I/O bottleneck is confirmed via `iostat`, and you trace it (via `lvs -o +devices`) to a specific physical disk. Before recommending hardware replacement/upgrade, what Day 1 concept would you specifically check to rule out a different, cheaper fix?**

I'd check whether this LV is striped across multiple PVs in a way that's concentrating load unevenly onto just this one specific physical disk rather than distributing it — if so, the "fix" might genuinely be a storage LAYOUT/configuration issue (rebalancing, or converting to a more evenly-distributed striping configuration) rather than the underlying hardware being fundamentally too slow for the aggregate workload. I'd also check whether this disk is part of a thin pool (Day 1, Topic 6) that might be experiencing additional overhead/contention from OTHER thin LVs sharing the same pool, in which case the actual fix might be addressing the pool's over-commitment or a different thin LV's unexpectedly heavy usage, rather than this specific LV's own workload being genuinely too much for the hardware — both of these represent genuinely different, often cheaper fixes than hardware replacement, and ruling them out first before recommending a hardware upgrade demonstrates the kind of full-stack reasoning (connecting Day 10's live diagnostic finding back to Day 1's storage architecture knowledge) that distinguishes a surface-level "iostat says this disk is busy, buy a new one" answer from a genuinely thorough one.

**5. How would you determine whether a reported "slow application" issue is actually a disk I/O problem versus a NETWORK problem, when the application itself both reads from local storage AND communicates with a remote database over the network?**

I'd run the disk-focused (`iostat -xz`) and network-focused (`sar -n DEV`, or the Day 2 connectivity/latency diagnostic toolkit — `mtr`, checking for packet loss/latency to the remote database specifically) diagnostics IN PARALLEL, specifically during the window the slowness is actually occurring, rather than guessing which one to check first based on assumption alone. Clean, low-latency, low-loss network diagnostics during the observed slow period would point toward the LOCAL disk I/O layer as the actual culprit (and I'd proceed with this topic's disk-specific escalation); conversely, elevated latency or loss specifically to the remote database correlating with the slow periods would point toward the network path instead (pivoting to Day 2's network escalation order) — the key discipline is gathering evidence for BOTH plausible categories simultaneously during the actual slow window, rather than committing to investigating just one based on an unverified assumption about which is more likely, since the application's dual dependency (local disk AND remote network) genuinely means either could be the real cause without further evidence to distinguish them.

---

## Topic 4: Package Management Deep Dive — Beyond `dnf install`

### Quick Review
- `dnf history` maintains a full transaction log; `dnf history undo <id>` can roll back a SPECIFIC transaction — a genuine safety net for a bad update or dependency change.
- GPG signature verification (`rpm --import`, and dnf's own automatic verification against imported keys) confirms package authenticity/integrity BEFORE installation — a real supply-chain security consideration, not just a formality.
- Module streams (`dnf module list`) let RHEL 8+ offer MULTIPLE versions of the same software (e.g., different PostgreSQL major versions) side by side, switchable per-system — solving a real problem the older, single-version repository model couldn't address cleanly.
- `rpm -V` (verify) compares installed files against the package's RECORDED checksums/permissions at install time — a genuine tamper/accidental-modification detection tool, distinct from anything `dnf`'s normal update-checking does.

### Quick Learning

Package management beyond the basics is genuinely relevant interview territory because it connects directly to real production concerns: can you safely roll back a bad update (`dnf history undo`), can you verify a package's authenticity before trusting it (GPG), can you detect if a system file has been tampered with or accidentally modified outside normal package management (`rpm -V`), and do you understand WHY module streams exist as a genuinely different mechanism than just "multiple versions in the repo" (the actual problem: a traditional single-version repo forces the ENTIRE system onto one version of a given package, but different applications on the SAME system might genuinely need different major versions of a shared dependency like PostgreSQL — module streams let that coexist safely).

**dnf history — a real safety net, not just a log:**
```
  dnf update mypackage
       │
       ▼
  Update introduces a REGRESSION — application breaks

  dnf history                          <- find the transaction ID
  ──────────────────────                for the problematic update
  ID | Command      | Date
  47 | update mypackage | 2026-09-20

  dnf history info 47                  <- see EXACTLY what this
                                          transaction changed —
                                          confirm it's the right one
                                          before rolling back

  dnf history undo 47                  <- reverts SPECIFICALLY this
                                          transaction — a targeted,
                                          surgical rollback, not a
                                          full system restore or a
                                          manual, error-prone attempt
                                          to remember and reinstall
                                          the previous version by hand
```

### Implementation (Learn by Applying)

**Scenario:** Practice the full package management safety toolkit — verifying a package's authenticity before installing, using transaction history to safely roll back a problematic update, and using `rpm -V` to detect an unauthorized/accidental file modification.

```bash
# GPG verification - confirm what keys are currently trusted, and what verification actually checks
rpm -qa gpg-pubkey* --qf '%{name}-%{version}-%{release} --> %{summary}\n'

dnf install -y httpd                  # A normal install already verifies against trusted, imported GPG keys automatically

dnf history
dnf history info $(dnf history | awk 'NR==3{print $1}')    # Inspect a recent transaction's actual detail
```

Practice a targeted rollback:
```bash
dnf install -y tree                    # Install something simple to demonstrate rollback against
dnf history                              # Note the transaction ID for this install
dnf history undo <that-id>                # Roll back SPECIFICALLY this transaction
rpm -q tree 2>&1                          # Confirm it's genuinely gone again
```

Practice file-integrity verification with `rpm -V`:
```bash
rpm -V httpd                            # Should show clean output (no discrepancies) on an untampered install

# Simulate an unauthorized modification
echo "unauthorized change" >> /etc/httpd/conf/httpd.conf
rpm -V httpd                             # Now shows a discrepancy specifically for httpd.conf — genuine tamper/change detection
```
Understand the output format: `rpm -V`'s output codes (`S` size differs, `M` mode/permissions differ, `5` MD5 checksum differs, `T` timestamp differs, and others) each indicate a SPECIFIC kind of deviation from the package's originally-recorded state — worth being able to read, not just recognize that SOME discrepancy exists.

Explore module streams conceptually:
```bash
dnf module list postgresql              # Shows MULTIPLE available major versions, side by side
dnf module info postgresql:13 2>/dev/null | head -10
# dnf module enable postgresql:13       # Would select this specific stream for this system,
                                           # distinct from a DIFFERENT system on the same RHEL release
                                           # choosing postgresql:15 instead — genuinely different
                                           # major versions coexisting across the RHEL ecosystem,
                                           # something a traditional single-version repo model
                                           # could never cleanly support
```

### Interview Questions — with Answers

**1. Explain what `dnf history undo <id>` actually does, and why it's a meaningfully better recovery mechanism than manually trying to reinstall a package's previous version by hand after a bad update.**

`dnf history undo` reverses a SPECIFIC, precisely-identified transaction — reverting exactly the packages, versions, and dependency changes that transaction originally made, using dnf's own recorded transaction history as the source of truth for what needs to be undone. Manually attempting to "reinstall the previous version" by hand requires you to correctly remember (or somehow determine) the exact previous version number, correctly identify every dependency that transaction might have also changed (not just the primary package itself), and manually reconstruct all of that — a genuinely error-prone process compared to dnf's own precise, automatic transaction-level tracking, which already has the complete, accurate record of exactly what changed and can reverse it deterministically without relying on human memory or reconstruction.

**2. Why does GPG signature verification matter for package installation as a genuine security consideration, not just a formality — walk through the actual attack scenario it protects against.**

GPG verification confirms that a package genuinely comes from the claimed, trusted source (the vendor/repository whose key you've imported and trust) and hasn't been tampered with in transit or by a compromised mirror/repository — without this verification, an attacker who compromises a repository mirror, performs a man-in-the-middle attack during package download, or otherwise manages to substitute a malicious package for a legitimate one could potentially get that malicious package installed with no indication anything was wrong, since a package's FILENAME and apparent metadata alone provide no cryptographic guarantee of authenticity. GPG verification specifically closes this gap — a package whose signature doesn't match a trusted, imported key is flagged/rejected, providing genuine protection against exactly this supply-chain-compromise attack scenario, not merely a bureaucratic formality with no real security function.

**3. What's the difference between what `dnf update` checks/does and what `rpm -V` checks — why might a system pass all its `dnf update` checks cleanly while still failing an `rpm -V` verification?**

`dnf update` is concerned with whether NEWER VERSIONS of installed packages are available in the configured repositories, and updating to them — it has no concern with whether the CURRENTLY installed files still match what was originally installed at their current version. `rpm -V` compares the CURRENTLY installed files' actual state (size, permissions, checksum, and more) against what the package database recorded as correct AT INSTALL TIME for the CURRENTLY installed version — these are answering fundamentally different questions. A system could be fully up to date (every package at its latest available version, `dnf update` reports nothing to do) while simultaneously having files that have been manually modified, corrupted, or tampered with SINCE that version was installed — `dnf update`'s version-currency check has no visibility into this at all, which is exactly why `rpm -V` exists as a genuinely separate, complementary integrity-verification tool rather than something `dnf update` already covers.

**4. Explain the actual problem module streams solve — why couldn't a traditional, single-version repository model handle the scenario module streams are designed for?**

Different applications on the SAME system, or different systems within the same organization on the same RHEL release, can genuinely need DIFFERENT major versions of the same underlying software (e.g., one application requiring PostgreSQL 13's specific behavior/features, while a different application or team needs PostgreSQL 15) — a traditional repository model offering exactly ONE version of a given package name per RHEL release forces every system to use that SAME single version, with no clean way to accommodate genuinely different major-version needs across different use cases within the same OS release's lifecycle. Module streams solve this by letting the repository offer MULTIPLE, independently-selectable major-version "streams" of the same software side by side — each system can explicitly choose (`dnf module enable`) which stream it wants, getting genuinely different major versions coexisting cleanly across the broader RHEL ecosystem without needing entirely separate OS releases or manual, unsupported version overrides to achieve this flexibility.

**5. A file managed by an RPM package shows a discrepancy in `rpm -V` output with the code `5` in the checksum column, but the size and permissions show no discrepancy at all. What does this specific pattern suggest, and what questions would it raise for further investigation?**

A checksum (`5`, indicating the MD5 digest differs) discrepancy with UNCHANGED size and permissions suggests the file's actual CONTENT has been modified in a way that doesn't change its overall size — ruling out a simple truncation or obvious large-scale change, and instead suggesting either a precise, deliberate content edit (someone specifically modifying configuration values within the file), or content substituted with something of coincidentally identical size. This pattern would raise the question of WHO made this change and WHY — was it a legitimate, deliberate configuration change made by an administrator (that simply wasn't tracked/documented properly, which is itself a process gap worth addressing) or something more concerning that warrants deeper investigation — checking file modification timestamps, correlating against known change windows/tickets, and reviewing audit logs (tying back to Day 4's auth/sudo audit trail concepts) for who actually had access to modify this specific file around the time the discrepancy likely originated, are the natural next investigative steps this specific, narrow discrepancy pattern would prompt.

---

## Topic 5 (Bonus — Advanced/Capstone): The Complete "Server Is Slow" Diagnostic Methodology

### Quick Review
- A genuinely complete performance diagnosis doesn't start with a tool — it starts with GATHERING THE ACTUAL COMPLAINT precisely (slow for whom, doing what, since when, compared to what baseline).
- The correct technical escalation order: broad signal (`top`) → category disambiguation (`vmstat`) → the RELEVANT specific-category deep dive (CPU: `mpstat`/`pidstat`; disk: `iostat`; network: Day 2's escalation; memory: swap activity) → correlate the finding against the relevant DEEPER architectural knowledge from earlier days (Day 1 storage layering, Day 3 cgroup limits, Day 6 firewall/network config, Day 7 SELinux, Day 8 NFS/time).
- `sar`'s historical data is what makes a POST-INCIDENT investigation possible at all, versus only being able to diagnose problems happening live, right now.
- The final step — often skipped — is confirming the identified fix actually RESOLVED the reported symptom, not just that some metric improved in isolation.

### Quick Learning

This capstone topic exists specifically to make explicit what every previous day's troubleshooting sections have been implicitly building toward: a genuinely senior diagnostic instinct isn't a memorized list of commands, it's a DISCIPLINED, ORDERED PROCESS that scales from "quick, obvious answer" through "deep, cross-day architectural correlation" as needed — and knowing when to stop escalating (because you've found and confirmed the actual cause) versus when a finding at one layer should prompt you to correlate against a COMPLETELY different day's knowledge (a disk bottleneck that's actually a Day 1 thin-pool issue; a network slowness that's actually a Day 8 stale/hung NFS mount; a "random" service failure that's actually a Day 3 cgroup memory limit being hit) is exactly the skill a product-company interview's open-ended "the server is slow, diagnose it" scenario is designed to probe.

**The complete methodology — gathering context, then escalating through categories, correlating outward as needed:**
```
  STEP 0: Gather the actual complaint PRECISELY
  ───────────────────────────────────────────────
  "Slow" for WHOM (one user, all users, a specific
  operation)? Since WHEN (sudden onset vs. gradual)?
  Compared to WHAT baseline (was it ever meaningfully
  faster, or is this actually the normal, expected
  performance for this workload)? — skipping this step
  and jumping straight to tools is a common mistake that
  wastes time investigating the wrong thing entirely.

  STEP 1: Broad, immediate signal
  ─────────────────────────────────
  top/htop, uptime — anything obviously abnormal at a glance?

  STEP 2: Disambiguate the CATEGORY
  ────────────────────────────────────
  vmstat 2 5 — CPU-bound? I/O-wait-bound? Swap-thrashing?
  (this single step determines EVERYTHING about which
   specific tools/direction to pursue next)

  STEP 3: Deep-dive the IDENTIFIED category specifically
  ──────────────────────────────────────────────────────
  CPU:      mpstat -P ALL, then pidstat -u  (Topic 2)
  Disk I/O: iostat -xz, then pidstat -d      (Topic 3)
  Network:  Day 2's FULL escalation order (ip/route/
            gateway/DNS/port/tcpdump), plus sar -n DEV
  Memory:   sustained si/so activity, then identify
            WHICH process via pidstat/ps, THEN correlate
            against Day 3 cgroup limits — is this process
            hitting a configured MemoryMax=?

  STEP 4: CORRELATE outward — is the technical finding
  actually explained by a DIFFERENT day's architecture?
  ──────────────────────────────────────────────────────
  Disk bottleneck on a specific PV → Day 1 (thin pool?
    striping? multipath path degraded?)
  Network slowness → Day 8 (is this actually a stale/
    hung NFS mount, not a "network" problem at all?)
  A process keeps dying under load → Day 3 (cgroup
    MemoryMax= being hit and OOM-killing it deliberately)
  Intermittent, hard-to-reproduce auth-adjacent slowness →
    Day 8 (clock drift affecting Kerberos specifically)

  STEP 5: CONFIRM the fix actually resolves the ORIGINAL
  reported complaint from Step 0 — not just that some
  metric moved in the right direction in isolation.
```

### Implementation (Learn by Applying)

**Scenario:** You receive a vague ticket: "the application is slow." Work the COMPLETE methodology end to end, including deliberately building in a case where the surface-level finding needs correlation against a specific earlier day's architectural knowledge to actually reach the true root cause — not just the first plausible-looking metric.

```bash
# STEP 0 (conceptual, but essential in a real scenario): before touching a single command,
# confirm specifics — this lab assumes we've established: "the reporting service's API
# responses have been slow for the last hour, previously fine, affects all users"

# STEP 1: broad signal
top -bn1 | head -15
uptime
```

```bash
# STEP 2: disambiguate category
vmstat 2 5
# Suppose this reveals: r modest, wa LOW, b modest — NOT CPU-bound, NOT classically I/O-wait-bound —
# genuinely ambiguous at this layer, which is itself useful information pointing elsewhere
```

```bash
# STEP 3: since CPU/disk don't show the classic signature, pivot toward network (Day 2's escalation)
# and toward checking if this app depends on NFS-mounted data (Day 8)
mount | grep nfs                        # Discover the application DOES have an NFS-mounted dependency
ls /mnt/app_data 2>&1                    # Test directly — does this hang, or return quickly?
timeout 5 ls /mnt/app_data; echo "exit: $?"   # If this HANGS (exit code from timeout, not a clean return)...
```

```bash
# STEP 4: CORRELATE outward — this points directly back to Day 8's stale-vs-hung NFS diagnosis,
# NOT a CPU/disk/network-in-the-generic-sense problem at all
rpcinfo -p nfsserver 2>&1 | head -5      # Confirm server-side reachability specifically
showmount -e nfsserver 2>&1

# If the server IS reachable but this specific mount is degraded/hanging intermittently,
# this could ALSO correlate back to Day 1's multipath knowledge if this NFS server's OWN
# backing storage is SAN-based and experiencing a degraded path:
multipath -ll 2>/dev/null | grep -i fault    # (run on the NFS SERVER itself, if accessible)
```

```bash
# STEP 5: after applying whatever fix the correlated root cause actually calls for
# (remounting a stale handle, force-unmounting a hung one, replacing a degraded multipath
# leg on the storage server), CONFIRM against the ORIGINAL complaint specifically:
time curl -s -o /dev/null -w "%{time_total}\n" http://localhost/api/health   # Actual app-level response time,
                                                                               # the thing that was ORIGINALLY reported
                                                                               # as slow — confirm THIS improved,
                                                                               # not just that some intermediate
                                                                               # metric looked better in isolation
```

### Interview Questions — with Answers

**1. Why is "gather the precise complaint before touching any diagnostic tool" listed as step ZERO of this methodology, rather than just starting directly with `top`?**

Jumping straight to tools without first understanding EXACTLY what "slow" means in this specific report — for whom, since when, compared to what — risks investigating a problem that doesn't actually match what's genuinely being experienced, wasting significant time chasing a plausible-looking but ultimately irrelevant metric. "Slow since this morning" versus "has always been somewhat slow, but someone just noticed today" are completely different investigations — the first points toward something that recently CHANGED, and I'd want to know what changed around that time; the second suggests either a long-standing, possibly-always-acceptable performance characteristic that's only now being questioned, or a very gradual degradation that wouldn't show up by comparing "before vs. after" a specific recent point in time at all. Getting this context precisely right BEFORE diagnosing avoids the common failure mode of confidently "fixing" a metric that was never actually the source of the reported problem.

**2. In the lab scenario, `vmstat` showed no classic CPU or disk I/O-wait signature at all. Why is this itself useful, actionable information, rather than a diagnostic dead end?**

A clean `vmstat` reading — no elevated run queue, no elevated I/O-wait, no swap activity — actively RULES OUT an entire category of potential causes (genuine CPU contention, genuine local disk I/O bottleneck, memory pressure/thrashing) with real confidence, which is just as valuable as a positive finding would have been, because it correctly redirects further investigation AWAY from those categories and toward the remaining plausible ones (network, or a dependency like NFS that doesn't necessarily show up as classic local I/O-wait in the same way). Treating a "clean" reading as a dead end, rather than as genuinely informative negative evidence that narrows the investigation, is exactly the kind of incomplete diagnostic thinking this methodology is designed to move past — every step, whether it finds something or rules something out, should actively inform what to check next.

**3. Explain why this methodology explicitly includes a step for correlating a technical finding against EARLIER DAYS' architectural knowledge, rather than stopping once you've identified the immediate technical symptom (e.g., "the mount is hanging").**

Identifying THAT something is hanging (a surface-level technical symptom) is necessary but not sufficient — it doesn't yet tell you WHY it's hanging, or what the actual, underlying, fixable root cause is; "the NFS mount is hanging" could stem from many different underlying causes (the NFS server process itself being down, a network path issue between client and server, or — as the lab specifically builds toward — the NFS server's OWN backing SAN storage experiencing a degraded multipath leg, a completely different layer of the stack entirely). Stopping at "the mount is hanging" without correlating further would likely lead to an incomplete or even entirely wrong fix (repeatedly remounting a client, when the actual problem is server-side storage degradation that a client-side remount does nothing to address) — the correlation step is what connects an observed SYMPTOM to its actual, addressable ROOT CAUSE, which often lives in a completely different architectural layer than where the symptom was first noticed.

**4. Why does the methodology insist on confirming the fix against the ORIGINAL reported complaint (Step 5), rather than considering the investigation complete once the identified technical root cause has been addressed?**

It's possible to correctly identify and genuinely fix A real technical issue that TURNS OUT not to have been the actual (or the ONLY) cause of the originally reported symptom — perhaps there were multiple contributing factors, or the identified and fixed issue was real but a secondary concern rather than the primary driver of the specific complaint that was reported. Confirming against the ORIGINAL complaint specifically (in the lab, the actual application-level API response time, not just "the NFS mount no longer hangs when I test it directly") is what verifies the investigation genuinely resolved what was ACTUALLY reported, rather than closing out a ticket having fixed A real problem that might not have been THE problem the user experienced — this discipline prevents a false sense of resolution where a genuinely-fixed technical issue doesn't actually translate into the reported symptom improving, which would otherwise only be discovered later when the original complaint resurfaces despite the "fix" having been applied.

**5. This entire day has been framed as a capstone connecting back to all nine previous days. Pick any TWO specific prior-day concepts (not already used in this topic's own lab) and describe a plausible "server is slow" scenario where correlating a Day 10 technical finding against THAT specific prior day's knowledge would be the key to reaching the true root cause.**

One example: a "server is slow" complaint traced via `vmstat`/`pidstat` to a specific application process being repeatedly killed and restarted under load — the surface Day 10 finding is "this process keeps dying," but correlating against Day 3's cgroup/systemd knowledge reveals the process is hitting a configured `MemoryMax=` limit and being deliberately OOM-killed within its own cgroup, not a mysterious crash — the actual fix is either legitimately raising the configured memory ceiling (if the workload's real needs have grown) or investigating why the process is consuming more memory than expected (a genuine leak), neither of which a purely Day-10-level "process died, check the logs" investigation would have connected to its actual, cgroup-level cause without that Day 3 correlation. A second example: intermittent, hard-to-reproduce slowness specifically affecting operations that involve authentication, correlating via Day 8's clock-drift knowledge — `vmstat`/`iostat`/network checks all come back clean because the actual root cause is Kerberos ticket validation intermittently failing/retrying due to clock skew between hosts, a root cause entirely invisible to Day 10's standard CPU/memory/disk/network toolkit and only reachable by recognizing the SPECIFIC symptom pattern (intermittent, auth-adjacent, no clean technical signature in the usual performance tools) that Day 8 specifically flagged as clock drift's signature.

---

**End of Day 10, and end of the core ten-day curriculum.** You should now be able to apply `tuned-adm` profiles as an evidence-informed starting point rather than a complete answer, work the correct CPU/memory bottleneck escalation to disambiguate genuine CPU contention from I/O-wait from swap-thrashing, diagnose disk and network I/O saturation with the right columns and the right historical tooling for post-incident investigation, use dnf/rpm's full safety toolkit (transaction rollback, GPG verification, file-integrity checking, module streams) beyond basic install/update, and — as the true capstone — run a complete, disciplined "server is slow" methodology that knows when to escalate deeper into a single category versus when to correlate a finding outward against the architectural knowledge from any of the previous nine days.

The remaining chapters — Installation & Patching, Server Hardening, RHEL/Distro Comparison, the Cloud/DevOps/SRE Bridge, and Study Resources — round out the guide beyond the daily study structure.
