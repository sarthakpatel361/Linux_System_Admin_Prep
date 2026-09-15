# Day 7 — SELinux (Full Study Edition, with Answers)

**Estimated time: 5–6 hours**

SELinux is the topic where product-company interviewers most reliably catch candidates who've only ever run `setenforce 0` to "fix" a problem. The entire point of today is building the discipline to never reach for that command as a solution — and to understand *why* a denial happened well enough to fix the actual context/boolean/policy issue instead of disabling the security layer that was doing its job correctly. This day covers modes, the type-enforcement model behind contexts, runtime booleans, the full denial-troubleshooting workflow, and — as the advanced capstone — how SELinux extends into container isolation via sVirt, which is directly relevant if your career is heading toward the cloud/SRE direction this guide is ultimately bridging to.

---

## Topic 1: SELinux Modes — Enforcing, Permissive, Disabled

### Quick Review
- **Enforcing** — policy is actively applied; violations are blocked AND logged.
- **Permissive** — violations are logged but NOT blocked; useful for testing/debugging policy without breaking anything.
- **Disabled** — SELinux is entirely off; no logging, no enforcement, and switching back on later requires a full filesystem relabel.
- `setenforce 0/1` changes the CURRENT, RUNTIME mode only; `/etc/selinux/config` controls what mode is set on the NEXT boot.

### Quick Learning

The critical distinction interviewers probe here isn't "what do the three modes do" — it's understanding that `setenforce 0` is a legitimate, useful DEBUGGING tool (temporarily permissive, so you can see what SELinux WOULD have blocked without it actually blocking anything, while you investigate) and a completely different thing from "the fix." Disabling SELinux, or leaving it permissive indefinitely as a permanent "solution" to some unexplained denial, is treated by essentially every serious product-company interviewer as a wrong answer — the correct instinct is always "identify and fix the specific context/boolean/policy gap," with permissive mode used only as a temporary diagnostic aid along the way.

**Enforcing vs. permissive — same denial, very different outcome:**
```
  ENFORCING mode                          PERMISSIVE mode
  ───────────────                          ─────────────────
  App tries an action SELinux             App tries an action SELinux
  policy doesn't allow                     policy doesn't allow
        │                                        │
        ▼                                        ▼
  Action is BLOCKED                        Action is ALLOWED to
  (app sees "permission denied")            proceed normally
        │                                        │
        ▼                                        ▼
  Denial is LOGGED to audit.log            Denial is STILL LOGGED
  (same as permissive — logging             to audit.log — this is
   happens either way)                      why permissive is useful
                                             for testing: you get the
                                             SAME diagnostic information,
                                             without breaking the app
                                             while you investigate

  DISABLED mode: no policy consulted at all, NOTHING logged, NOTHING
  enforced — this is qualitatively different from permissive, not
  just "more relaxed" — you lose ALL visibility, not just enforcement
```

### Implementation (Learn by Applying)

**Scenario:** You're investigating a suspected SELinux-related application failure. Use permissive mode correctly as a diagnostic tool — confirming the hypothesis without permanently weakening the system's security posture — then restore enforcing mode and fix the actual underlying issue.

```bash
getenforce
sestatus                              # More detail: mode, policy type, mount point, etc.

# Reproduce a real denial: try serving web content from a non-standard, unlabeled directory
mkdir -p /custom_webroot
echo "test page" > /custom_webroot/index.html
chcon --reference=/var/www/html /custom_webroot 2>/dev/null || true   # Deliberately skip proper context for now
# (Apache/httpd would be configured to serve from here — denial expected)

ausearch -m avc -ts recent            # Check for a fresh denial
```

Use permissive mode specifically as a diagnostic tool, temporarily:
```bash
setenforce 0
getenforce                            # Confirm: Permissive

# Retry the action that was failing — it now SUCCEEDS, confirming SELinux was the actual blocker
# (this doesn't mean "leave it here" — it CONFIRMS the hypothesis so you can fix it correctly)

setenforce 1                          # Restore enforcing immediately once the hypothesis is confirmed
getenforce
```

Persist the CORRECT fix (proper context, covered fully in Topic 2) rather than leaving the system permissive:
```bash
semanage fcontext -a -t httpd_sys_content_t "/custom_webroot(/.*)?"
restorecon -Rv /custom_webroot
ls -Z /custom_webroot
```

### Interview Questions — with Answers

**1. Why is `setenforce 0` considered a legitimate diagnostic step, but leaving a production system in permissive mode indefinitely considered a security anti-pattern?**

`setenforce 0` is legitimate as a TEMPORARY diagnostic tool because permissive mode still logs every denial exactly as enforcing mode would — using it briefly lets you confirm "is SELinux actually the cause of this problem" and gather the exact denial details needed to craft a correct fix, all without permanently disabling the security boundary. Leaving a system permissive indefinitely is a real problem because it means SELinux's actual enforcement — the thing that would BLOCK a genuinely malicious or compromised process from doing something it shouldn't — is switched off entirely, even though the system LOOKS like it has SELinux "enabled" (permissive is still technically "on," which is exactly why it's a dangerous middle state that can go unnoticed in an audit if someone only checks whether SELinux is installed/loaded rather than checking the actual mode).

**2. What's the practical operational difference between switching to `disabled` mode versus leaving a system in `permissive` mode — beyond "one enforces and one doesn't"?**

Permissive mode still actively evaluates every action against policy and logs every violation that WOULD have been blocked — you retain full visibility into what SELinux thinks is happening, even without enforcement. Disabled mode means SELinux isn't consulted AT ALL — no policy evaluation, no logging, nothing; you lose all visibility, not just enforcement. This distinction matters practically because a system switched to disabled mode, if later switched back to enforcing, will have a filesystem full of files with NO SELinux labels applied (since nothing was labeling anything while disabled) — requiring a full filesystem relabel (`touch /.autorelabel && reboot`, or `restorecon -R /`) before enforcing mode can work correctly again, a disruptive operation permissive mode never requires since labeling continued happening normally throughout.

**3. A colleague suggests permanently disabling SELinux on a new production server "to avoid dealing with denials" during initial application deployment. How would you respond, and what would you propose instead?**

I'd push back on permanently disabling it — that removes a meaningful security layer for the system's entire lifecycle just to avoid some upfront investigation work during a one-time deployment phase, which is a poor long-term tradeoff. I'd propose instead: deploy with SELinux in permissive mode specifically during the initial rollout, actively monitoring `ausearch -m avc` or `sealert` for denials as the application runs through its real workflows, using that window to identify and properly fix every genuine context/boolean gap the application needs — then switch to enforcing mode once the denial log is clean, having done the actual work of understanding and correctly configuring the application's real SELinux requirements rather than skipping that understanding entirely.

**4. If a system has been running in `disabled` mode for an extended period and you need to switch it back to `enforcing`, what extra step is required that wouldn't be needed coming from `permissive` mode, and why?**

A full filesystem relabel is required — typically via `touch /.autorelabel` followed by a reboot (which triggers the relabel automatically early in the boot process), or `restorecon -R /` run directly. This is necessary because while SELinux was disabled, no process was maintaining or applying SELinux labels to files as they were created, modified, or moved — the filesystem's actual SELinux context labels can be stale, missing, or simply wrong relative to what the currently-loaded policy expects. Coming FROM permissive mode doesn't need this, because permissive mode still actively applies and maintains correct labeling throughout — only actual ENFORCEMENT was skipped, not the labeling process itself, whereas disabled mode skips both.

**5. Explain `sestatus` output versus `getenforce` — when would you need the fuller detail `sestatus` provides rather than just the quick mode check?**

`getenforce` gives you a single-word answer: the current mode (Enforcing/Permissive/Disabled), nothing more. `sestatus` gives a fuller picture: current mode, the mode configured for the NEXT boot (which can differ from the current running mode if someone used `setenforce` without updating `/etc/selinux/config`), the loaded policy type (targeted vs. mls, most systems use targeted), the policy version, and the SELinux filesystem mount point. I'd reach for the fuller `sestatus` output specifically when troubleshooting a discrepancy between expected and actual behavior across a reboot — for example, confirming whether a `setenforce` change made live will actually PERSIST through the next reboot, which `getenforce` alone can't tell you since it only reports the current, possibly-temporary runtime state.

---

## Topic 2: Contexts and Type Enforcement — What's Actually Being Checked

### Quick Review
- An SELinux context is `user:role:type:level` — for the vast majority of practical troubleshooting, the **type** field is what actually drives access decisions.
- `ls -Z` shows file contexts; `ps -Z` shows process contexts.
- `chcon` changes a context TEMPORARILY — lost on the next relabel/`restorecon`. `semanage fcontext` writes a PERSISTENT rule that survives relabeling.
- Type Enforcement is fundamentally about matching a process's DOMAIN (its type) against the TYPE of the resource it's trying to access — a policy either allows or denies that specific domain-to-type interaction.

### Quick Learning

Understanding SELinux at the "it's like extra permissions" level is exactly the shallow understanding interviewers probe past. The real model is **Type Enforcement**: every process runs within a security DOMAIN (itself just a type, applied to processes), every resource (file, port, etc.) has a TYPE, and policy defines which domains are allowed which specific operations against which specific types — httpd's domain being allowed to read files of type `httpd_sys_content_t` is a specific, named policy rule, not a general "web server stuff" allowance. This is why a perfectly permission-correct file (readable by everyone at the Unix level) can still be denied to a process — the TYPE doesn't match what that process's domain is permitted to touch, regardless of standard rwx bits agreeing.

**Type enforcement — two separate, independently-checked permission systems:**
```
  Standard Unix permissions           SELinux Type Enforcement
  ──────────────────────────          ─────────────────────────
  "Can THIS USER read this file       "Can a process running in
   based on rwx bits?"                 THIS DOMAIN access a resource
                                        of THIS TYPE, per policy?"

  chmod 644 /custom_webroot/index.html   ls -Z /custom_webroot/index.html
  (world-readable at the Unix level)     unconfined_u:object_r:default_t:s0
                                                              ▲
                                                              WRONG type —
                                                              httpd's domain
                                                              (httpd_t) has
                                                              no policy rule
                                                              allowing it to
                                                              read default_t

  BOTH checks must independently PASS for access to succeed.
  Unix permissions being wide open does NOT override or bypass
  SELinux denying the SAME access based on a type mismatch —
  they are two entirely separate, independently-enforced systems.
```

### Implementation (Learn by Applying)

**Scenario:** Prove to yourself, hands-on, that Unix permissions and SELinux contexts are genuinely independent systems — build a file that's completely open at the Unix level, yet still blocked by SELinux, then fix it at the correct layer.

```bash
mkdir -p /custom_webroot2
echo "test" > /custom_webroot2/index.html
chmod -R 777 /custom_webroot2          # Maximally permissive at the Unix level — deliberately, to isolate the SELinux variable

ls -Z /custom_webroot2/index.html      # Check the actual TYPE — likely default_t or similar, NOT httpd_sys_content_t
ls -Z /var/www/html/                   # Compare against the correctly-labeled default web directory's type

# Configure httpd (conceptually) to serve from /custom_webroot2, then attempt access
# Even with 777 permissions, httpd's domain (httpd_t) will be denied access to default_t content
ausearch -m avc -ts recent | grep webroot2 2>/dev/null
```

Fix it at the CORRECT layer — persistent context, not Unix permissions (which were never the actual problem):
```bash
semanage fcontext -a -t httpd_sys_content_t "/custom_webroot2(/.*)?"
restorecon -Rv /custom_webroot2
ls -Z /custom_webroot2/index.html      # Now correctly labeled httpd_sys_content_t
```

Demonstrate the chcon-vs-semanage persistence difference directly:
```bash
chcon -t httpd_sys_content_t /custom_webroot2/index.html   # Temporary — works right now
restorecon -v /custom_webroot2/index.html                   # Simulates a relabel event
ls -Z /custom_webroot2/index.html                            # If semanage rule wasn't ALSO set, this would REVERT
                                                                # to whatever restorecon considers "default" for this path
```

### Interview Questions — with Answers

**1. Explain why a file with `chmod 777` permissions can still be denied access by a process, using the actual mechanism — not just "SELinux is stricter."**

Standard Unix permissions and SELinux Type Enforcement are two entirely separate, independently-evaluated access control systems checked sequentially — passing one does not exempt you from the other, and both must independently allow the access for it to succeed. `chmod 777` makes a file maximally open at the Unix permission layer (any user, any operation), but SELinux separately checks whether the TYPE of that file is one the requesting process's DOMAIN has an explicit policy rule permitting it to access — if the file's type (say, `default_t`, an unremarkable generic type) isn't one that domain (say, `httpd_t`) has any policy rule allowing interaction with, the request is denied regardless of how permissive the Unix-level bits are, because SELinux never even considers the Unix permission bits as relevant to ITS decision.

**2. What's the actual difference between `chcon` and `semanage fcontext -a`, and describe the specific failure mode of using `chcon` when a persistent fix was actually needed.**

`chcon` directly and immediately changes a file's current SELinux context, but this change lives only in the file's actual, current metadata — it has no awareness of or interaction with SELinux's underlying POLICY about what context that path SHOULD have. `semanage fcontext -a` instead writes a persistent RULE into the policy itself, associating a path pattern with a specific context — this rule is what tools like `restorecon` consult to know the "correct" context for a given path. The failure mode: if you use `chcon` alone (without a corresponding `semanage fcontext` rule) and later something triggers a relabel — either an explicit `restorecon` run, or a full filesystem relabel — the context reverts back to whatever the POLICY says it should be (which, without your `semanage` rule, is likely NOT what you set via `chcon`), silently undoing your fix at a moment that can be confusing and hard to immediately connect back to "oh, I used chcon instead of semanage months ago."

**3. You're troubleshooting a denial and `ls -Z` shows a file's context, but you're not sure which PART of that context (user, role, type, or level) is actually relevant to the denial. How do you know, and why does that simplification matter practically?**

For the overwhelming majority of practical SELinux troubleshooting on a standard `targeted` policy system (the default on RHEL), the TYPE field is what drives Type Enforcement access decisions — the user, role, and level (MCS/MLS) fields exist and matter in more complex, specialized configurations (like multi-level security environments or SELinux users mapped to Linux users for role-based access), but for the vast majority of everyday service-access denials (a web server, a database, a custom application), the type mismatch is the thing to focus on. This simplification matters practically because it tells you where to focus your troubleshooting attention first — checking `semanage fcontext -l` for existing type rules and comparing types via `ls -Z`/`ps -Z`, rather than getting lost trying to reason about all four context fields with equal weight when three of them are rarely the actual cause of a typical denial.

**4. Explain what "domain" means in the context of SELinux Type Enforcement, using `ps -Z` output as a concrete illustration.**

A domain is simply a TYPE applied to a running PROCESS rather than to a file or other resource — `ps -Z` shows this directly, e.g., an httpd process might show a context ending in `httpd_t`, which IS its domain. Type Enforcement policy is fundamentally a set of rules defining which domains are permitted which specific operations (read, write, execute, connect, etc.) against which specific resource types — so "httpd_t can read httpd_sys_content_t files" is a concrete, specific policy rule connecting a process domain to a resource type, and this domain-to-type relationship, checked against loaded policy, is the actual mechanism underlying every SELinux access decision for that process.

**5. A newly deployed application writes its data files to a custom directory, and despite the directory having a correctly-set `semanage fcontext` rule and a successful `restorecon`, the application STILL gets denied. What would you check next, given everything covered so far?**

I'd check whether the DENIAL is actually about the SAME operation/type interaction the context rule was meant to fix, or a DIFFERENT one entirely — `semanage fcontext`/`restorecon` fixes file/directory labeling, but the actual denial might be about something else, such as a network-related SELinux boolean the application also needs (covered in Topic 3), or the application needing to connect to a port type it isn't currently permitted to bind/connect to, or even a completely separate denial for a different file/operation than the one just fixed. I'd re-run `ausearch -m avc -ts recent` (or check `sealert` again) specifically to see the CURRENT, fresh denial details rather than assuming it's the same issue already addressed — SELinux denials often come in a sequence, where fixing one reveals the NEXT thing the application needs, rather than one single denial explaining the entire problem.

---

## Topic 3: Booleans — Runtime Policy Toggles

### Quick Review
- Booleans are pre-defined, named ON/OFF switches for COMMON policy variations — no custom policy authoring required for well-known scenarios.
- `getsebool -a` lists every boolean and its current state; `setsebool` changes it.
- `setsebool boolname on` is runtime-only; `setsebool -P boolname on` is persistent across reboots.
- Booleans exist specifically because certain policy variations (like "should httpd be allowed to make outbound network connections") are common enough and legitimate enough in SOME deployments that Red Hat pre-built a clean, safe toggle rather than requiring every admin to author custom policy for the same recurring need.

### Quick Learning

Booleans are the SELinux escape hatch you should reach for BEFORE ever considering `audit2allow`-generated custom policy — if a common scenario has a named boolean, using it is both easier and safer than writing custom policy, because the boolean was specifically designed, scoped, and vetted by the policy's authors for exactly that use case, rather than being an auto-generated permission grant based on whatever a specific denial happened to ask for (which is `audit2allow`'s much blunter, historically over-permissive approach, covered in Topic 4).

**Booleans as pre-built, safe toggles for common scenarios:**
```
  getsebool -a | grep httpd
  ──────────────────────────
  httpd_can_network_connect --> off      <- "should httpd be allowed
  httpd_can_sendmail --> off                to make OUTBOUND network
  httpd_enable_homedirs --> off              connections at all?" — a
  httpd_use_nfs --> off                      SPECIFIC, well-understood,
                                              legitimately-sometimes-needed
                                              policy variation, pre-built
                                              as a clean toggle

  A web app needs to call an external API ⇒ needs outbound network
  access ⇒ this is EXACTLY the scenario httpd_can_network_connect
  exists for ⇒ setsebool -P httpd_can_network_connect on

  This is a MUCH safer, more targeted fix than:
  - setenforce 0 (disables ALL enforcement, system-wide)
  - audit2allow custom policy (grants EXACTLY what the specific
    denial asked for, which might be broader or narrower than
    the actual, well-reasoned intent behind the boolean)
```

### Implementation (Learn by Applying)

**Scenario:** A web application needs to make outbound API calls to an external service — a genuinely common, legitimate requirement that SELinux blocks by default. Diagnose the denial, recognize it as a boolean-shaped problem (not a context problem), and fix it correctly.

```bash
getsebool httpd_can_network_connect

# Simulate the denial (conceptually — httpd attempting an outbound connection while the boolean is off)
ausearch -m avc -ts recent | grep -i "name_connect\|httpd"
sealert -a /var/log/audit/audit.log 2>/dev/null | grep -A5 "network_connect" 
```
`sealert`'s output for this exact scenario typically SUGGESTS the boolean directly — one of SELinux's genuinely helpful features is that its denial analysis often correctly identifies "this looks like a case a known boolean addresses" rather than leaving you to guess.

```bash
setsebool -P httpd_can_network_connect on
getsebool httpd_can_network_connect        # Confirm: on, and persistent (survives reboot due to -P)
```

Explore the broader boolean landscape to build a mental map of what's commonly toggled:
```bash
getsebool -a | grep -c "off$"       # See how many booleans exist in their default (usually restrictive) state
getsebool -a | grep -i nfs           # e.g., relevant when you reach the NFS topic later in this guide
semanage boolean -l | grep httpd_can_network_connect   # -l shows the boolean's DESCRIPTION, useful when the name alone isn't self-explanatory
```

### Interview Questions — with Answers

**1. Why should you generally check for an applicable boolean BEFORE reaching for `audit2allow` to generate custom policy for a denial?**

Booleans represent pre-built, deliberately-scoped, well-tested policy variations for known, common scenarios — when one exists for your exact situation, it's both easier (a single `setsebool` command versus authoring and loading a custom policy module) and safer (the boolean's scope was carefully designed by the policy's actual authors for this specific use case, rather than being auto-generated based purely on whatever a particular denial happened to request, which can be broader or narrower than truly necessary). Reaching for `audit2allow` first, when a targeted boolean would have solved the exact same problem more precisely, risks creating unnecessarily broad or oddly-scoped custom policy when a clean, vetted, standard solution was already available.

**2. What's the practical difference between `setsebool boolname on` and `setsebool -P boolname on`, and describe a real scenario where forgetting `-P` causes a confusing problem later.**

Without `-P`, the boolean change applies immediately to the CURRENT runtime state but is not persisted — it reverts to its previous (usually off/default) value on the next reboot. With `-P`, the change is written to persistent policy configuration and survives reboots. A real scenario: an admin diagnoses and fixes an application's SELinux issue by toggling a boolean live (without `-P`) during a troubleshooting session, confirms the application now works, and closes the ticket — weeks later, the server reboots for routine patching, the boolean silently reverts to its default state, and the EXACT SAME "mysteriously broken" symptom reappears, confusing whoever's on call at the time since from their perspective nothing was changed recently — the actual root cause (a non-persistent boolean fix from weeks ago) is not obviously connected to a routine reboot unless you specifically know to check for this pattern.

**3. `getsebool -a` shows dozens or hundreds of booleans, many with names that aren't immediately self-explanatory. How would you find out what a specific boolean actually controls before deciding whether to enable it?**

`semanage boolean -l` lists booleans along with their actual DESCRIPTION text, which explains in plain language what enabling that specific boolean permits — this is significantly more informative than the boolean's name alone, which can sometimes be ambiguous or assume context you don't have. I'd always check the description before enabling an unfamiliar boolean, specifically to understand the actual scope of what I'm permitting, rather than enabling something based on a name that SOUNDS right without confirming what it genuinely does — enabling an overly-broad or wrong boolean is itself a real, if less common, way to weaken security posture through misunderstanding rather than deliberate custom-policy authoring.

**4. Why do booleans exist at all as a distinct mechanism, rather than SELinux policy just being a single, fixed set of rules with no runtime-toggleable variations?**

Different deployments of the SAME service can have genuinely different, equally legitimate requirements — one httpd deployment might be a purely static content server with zero need for outbound network access, while another is running an application that legitimately needs to call external APIs; a single, fixed policy would have to pick ONE of these behaviors as "correct" for all httpd deployments, forcing everyone in the other camp to either accept unwanted restriction or resort to broader custom policy. Booleans exist specifically to let policy express "here are the KNOWN, common variations this service might legitimately need, cleanly toggle-able per-deployment," accommodating genuinely different, equally valid production requirements without requiring every differently-configured deployment to author its own custom policy from scratch for what is, in reality, a well-understood and common variation.

**5. A boolean you've enabled with `-P` doesn't seem to actually be having any effect on the observed denial — the exact same denial keeps appearing in the audit log even after enabling it. What would you check?**

First, I'd confirm the boolean was actually correctly identified as relevant to THIS specific denial in the first place — `sealert`'s suggestion (or my own interpretation of the `ausearch`/audit log details) might have been for a different, similar-looking but distinct denial than the one still occurring; I'd re-examine the exact denial details (source type, target type, specific permission being denied) against exactly what the boolean's description says it controls, to confirm they genuinely match. If they do match and it's still not working, I'd verify the boolean actually took effect (`getsebool boolname` to confirm current state is genuinely "on," not just that the `setsebool` command appeared to succeed), and consider whether there might be a SEPARATE, additional denial also blocking the same overall operation — as noted in Topic 2's final question, SELinux denials often come in a sequence, and fixing one boolean-addressable denial can simply reveal the next distinct thing still being blocked.

---

## Topic 4: Denial Troubleshooting — The Complete, Correct Workflow

### Quick Review
- The correct order: **identify** the denial (`sealert`/`ausearch`) → **fix** via the most targeted appropriate mechanism (`restorecon`/`semanage fcontext`/`setsebool -P`) → only use `audit2allow` custom policy modules as a genuine last resort.
- `sealert -a /var/log/audit/audit.log` gives human-readable analysis AND suggested fixes — the correct starting point, not raw log reading.
- `audit2allow` custom modules can be dangerously broad if generated carelessly from a wide time range of unrelated denials — always scope the input precisely.
- `setenforce 0` as "the fix" is the single most common wrong answer interviewers listen for.

### Quick Learning

Everything in Topics 1-3 feeds into this: the correct workflow is a specific, ordered decision tree, not a grab-bag of tools to try randomly. Start with `sealert` for human-readable analysis (it often directly suggests the correct fix, including telling you when a boolean is the answer). If it's a labeling issue, fix it with `semanage fcontext` + `restorecon`. If it's a known-variation issue, fix it with `setsebool -P`. Only when NEITHER of those genuinely applies — a genuinely novel, application-specific access pattern with no existing boolean or simple context fix — does `audit2allow`-generated custom policy become the appropriate tool, and even then, scoped as narrowly and deliberately as possible.

**The complete decision tree, in the order it should actually be applied:**
```
  Denial occurs
       │
       ▼
  sealert -a /var/log/audit/audit.log    <- ALWAYS start here, not raw log grep
       │
       ▼
  Does sealert suggest a boolean?  ──YES──▶  setsebool -P <bool> on    [DONE]
       │ NO
       ▼
  Is this a LABELING issue (wrong type on a file/directory)?
       │
      YES──▶  semanage fcontext -a -t <type> "path"
               restorecon -Rv path                       [DONE]
       │ NO
       ▼
  Genuinely novel, app-specific access pattern with NO
  existing boolean or context fix available?
       │
      YES──▶  audit2allow -a -M custommodule (SCOPED to
               ONLY the specific, relevant denial(s) —
               never generated from a broad, mixed time
               range of unrelated denials)
               semodule -i custommodule.pp                [DONE, but
                                                             treat as
                                                             last resort]

  setenforce 0 as "the fix" is NEVER a terminal node in
  this decision tree — it's only ever the temporary,
  diagnostic detour used WHILE navigating the above,
  covered fully in Topic 1.
```

### Implementation (Learn by Applying)

**Scenario:** Work a denial through the complete, correct workflow end to end — including deliberately practicing the wrong instinct first (to recognize why it's wrong), then doing it correctly, and finally practicing the genuine last-resort custom-policy path with appropriate scoping discipline.

```bash
# Generate a realistic denial scenario: custom port for a service
semanage port -l | grep http_port_t     # See what ports httpd's domain is ALREADY allowed to bind

# Attempt to run httpd on a non-standard port not covered by existing policy (e.g., 8090)
ausearch -m avc -ts recent | grep 8090
sealert -a /var/log/audit/audit.log 2>/dev/null | grep -B2 -A10 "8090"
```

The WRONG instinct, demonstrated so you recognize it (do not stop here):
```bash
setenforce 0     # "Fixes" it by disabling enforcement entirely — WRONG, not a real fix, do not stop here
setenforce 1     # Restore immediately
```

The CORRECT fix for a port-type denial specifically:
```bash
semanage port -a -t http_port_t -p tcp 8090
semanage port -l | grep 8090            # Confirm the new port is now correctly associated with http_port_t
```

Practice the genuine last-resort path, with correct scoping discipline:
```bash
# ONLY after confirming no boolean or context/port fix applies — a genuinely novel access need
ausearch -m avc -ts recent -su system_u 2>/dev/null | audit2allow -M custom_app_policy

cat custom_app_policy.te                # ALWAYS review the generated .te file before loading it —
                                          # confirm it's scoped to EXACTLY the intended access, nothing broader
semodule -i custom_app_policy.pp
semodule -l | grep custom_app_policy    # Confirm it loaded
```

### Interview Questions — with Answers

**1. Walk me through the complete, correct decision tree for handling an SELinux denial, from the moment you first see it to the moment it's genuinely resolved.**

Start with `sealert -a /var/log/audit/audit.log` for human-readable analysis, which often directly suggests the correct fix category. If it points to a boolean, use `setsebool -P` — the most targeted, safest fix for a known common variation. If it's a labeling/context mismatch, fix it with `semanage fcontext -a` plus `restorecon` for a persistent, correct relabel. If neither applies — a genuinely novel, application-specific access requirement with no existing boolean or straightforward context fix — only then generate a NARROWLY SCOPED custom policy module via `audit2allow`, reviewing the generated `.te` file before loading it to confirm it grants exactly the intended access and nothing broader. `setenforce 0`/disabling SELinux is never a terminal step in this process — at most, it's a temporary diagnostic aid used to confirm SELinux is genuinely the cause before working through the actual fix.

**2. Why is generating an `audit2allow` custom policy module from a broad, unscoped time range of the audit log considered dangerous, compared to scoping it tightly to one specific, understood denial?**

A broad, unscoped `audit2allow` run picks up EVERY denial that occurred within that time range — potentially including unrelated denials from other processes, other legitimate security events that SHOULD stay blocked, or even denials generated by something actually malicious probing the system, if such activity happened to occur in that window. Generating custom policy from that broad set risks creating a module that grants access far beyond what was actually intended or needed — a genuinely over-permissive policy module masquerading as a fix for one specific, understood problem, precisely the kind of security-weakening custom policy that gives `audit2allow`-generated modules a bad reputation when used carelessly. Scoping tightly to one specific, already-understood denial (using precise `ausearch` filters by time, process, or other criteria) ensures the resulting policy module grants exactly the narrow access actually intended.

**3. A colleague generates and loads an `audit2allow` custom policy module without reviewing the generated `.te` file first, just running the suggested commands directly. What's the risk, and what should the correct habit be instead?**

The risk is exactly what question 2 describes — loading an unreviewed module means you have no actual confirmation of what access it's granting; it might be precisely scoped and correct, or it might be far broader than intended due to unrelated denials being swept into the same generation, and you'd have no way of knowing without having looked. The correct habit is ALWAYS reviewing the generated `.te` file's actual `allow` rules before running `semodule -i` to load it — confirming each rule genuinely matches the specific, understood access need you're trying to solve, and treating any rule that looks broader or unrelated to your actual problem as a signal to re-scope the `audit2allow` input more precisely (narrower time range, specific process filter) rather than blindly trusting the auto-generated output.

**4. Explain why `sealert`'s suggested fix should generally be your starting point rather than manually interpreting raw `ausearch`/audit.log output yourself.**

`sealert` performs its own analysis of the raw denial data and translates it into human-readable explanation, often including a SPECIFIC suggested remediation — correctly identifying, for many common scenarios, whether the issue is boolean-shaped, context-shaped, or something else, saving the manual interpretation work of parsing raw AVC denial syntax and cross-referencing it against policy documentation yourself. This doesn't mean blindly trusting `sealert`'s suggestion without understanding WHY it's suggesting that fix (you should still understand the underlying reasoning, which is exactly what this entire day has been building toward), but as a starting point for efficiently triaging "what kind of fix does this denial likely need," it's a faster, more reliable first step than manually parsing raw log output from scratch every time.

**5. If a denial's `sealert` analysis doesn't suggest ANY of the standard fixes (no boolean, no obvious context fix) and you genuinely need custom policy, what specific precautions would you take before deploying that custom module to production?**

I'd generate the module from the most narrowly scoped `ausearch` input possible (specific time range immediately around the confirmed, understood denial, filtered by the specific process/type involved if possible), then thoroughly review the resulting `.te` file's actual allow rules line by line, confirming each one corresponds to a legitimate, expected, and understood access need rather than accepting the auto-generated output wholesale. I'd test the loaded module thoroughly in a non-production environment first if at all possible, confirming it resolves the intended denial WITHOUT granting anything unexpectedly broader, and I'd document exactly why this custom module exists and what specific access gap it addresses — since custom policy modules are exactly the kind of thing that, without documentation, becomes a mysterious "why do we have this" finding for whoever audits the system's SELinux configuration months or years later.

---

## Topic 5 (Bonus — Advanced): sVirt and Container SELinux Contexts

### Quick Review
- **sVirt** extends SELinux's Type Enforcement model to provide isolation between virtual machines and, critically, between CONTAINERS running on the same host.
- Each container typically gets a unique, randomly-generated **MCS (Multi-Category Security) label** — even containers sharing the same base type are isolated from EACH OTHER via distinct category sets.
- This is why a container escape/compromise that somehow gained root-equivalent access WITHIN one container still can't necessarily read/write another container's files on shared underlying storage — SELinux's category-based isolation, not just standard Unix UID/namespace isolation, is providing an additional barrier.
- `ls -Z` on container-related files shows the actual MCS category labels in the context's LEVEL field — the part of the context largely ignored for standard troubleshooting in Topics 1-4, but centrally relevant here.

### Quick Learning

This bonus topic exists specifically because your career direction (per this guide's later cloud/SRE-bridging chapters) means container-adjacent SELinux knowledge is increasingly relevant even for a traditional Linux SysAdmin background — and it's also where the LEVEL field of the SELinux context (the part largely glossed over in Topics 1-4 as "mostly irrelevant for standard troubleshooting") suddenly becomes the actually-relevant part. sVirt reuses SELinux's existing Type Enforcement infrastructure but adds MCS category-based separation specifically so that multiple containers, which might all share broadly similar container-runtime process types, are still meaningfully isolated from EACH OTHER, not just from the host — this is an important layer of defense-in-depth beyond what Linux namespaces and cgroups (covered on Day 3) provide on their own.

**MCS categories — isolating containers from EACH OTHER, not just from the host:**
```
  Two containers, same base container runtime TYPE, but
  DIFFERENT randomly-assigned MCS category sets:

  Container A process:  system_u:system_r:container_t:s0:c123,c456
  Container B process:  system_u:system_r:container_t:s0:c789,c101

  Container A's files:  system_u:object_r:container_file_t:s0:c123,c456
  Container B's files:  system_u:object_r:container_file_t:s0:c789,c101

  Even though BOTH containers run under the SAME type (container_t)
  and would, by TYPE ENFORCEMENT alone, appear to have identical
  policy permissions to container_file_t resources generally —
  the MCS category sets (c123,c456 vs c789,c101) are DIFFERENT,
  and SELinux's MCS enforcement requires a MATCHING category set
  for access, meaning Container A's process literally cannot access
  Container B's files even if it somehow escaped its namespace
  isolation, because the CATEGORY LABELS don't match — a completely
  separate, additional barrier beyond namespace/cgroup isolation.
```

### Implementation (Learn by Applying)

**Scenario:** Run two containers on the same host and directly observe sVirt's MCS category isolation in action — confirming, hands-on, that they're isolated from each other at the SELinux layer, not merely by container runtime namespacing alone.

```bash
dnf install -y podman

# Run two simple containers and inspect their actual SELinux contexts
podman run -d --name container_a alpine sleep 3600
podman run -d --name container_b alpine sleep 3600

# Inspect the actual running processes' SELinux contexts on the HOST
ps -eZ | grep -E "container_a|container_b" 2>/dev/null
podman inspect container_a --format '{{.ProcessLabel}}'
podman inspect container_b --format '{{.ProcessLabel}}'
```
Note the differing MCS category numbers (the `c###,c###` portion) between the two containers' process labels, despite both sharing the same base container type.

```bash
# Inspect the containers' file storage on the host and confirm distinct category labels there too
podman inspect container_a --format '{{.GraphDriver.Data.UpperDir}}' 2>/dev/null
ls -Z $(podman inspect container_a --format '{{.GraphDriver.Data.UpperDir}}' 2>/dev/null) 2>/dev/null | head -3
ls -Z $(podman inspect container_b --format '{{.GraphDriver.Data.UpperDir}}' 2>/dev/null) 2>/dev/null | head -3
```

Clean up:
```bash
podman rm -f container_a container_b
```

### Interview Questions — with Answers

**1. Explain what problem sVirt's MCS category-based isolation solves that standard container namespace/cgroup isolation (from Day 3) does NOT fully address on its own.**

Namespace and cgroup isolation (process IDs, network, mount points, resource limits) provide strong isolation for NORMAL container operation, but they rely on the kernel's namespace/cgroup boundaries being correctly enforced and not bypassed — a sufficiently severe container escape vulnerability could potentially breach that boundary, at which point a compromised container process might attempt to access resources belonging to OTHER containers or the host directly. sVirt's MCS category labeling provides an ADDITIONAL, independent layer of defense specifically for this scenario: even if namespace isolation were somehow bypassed, SELinux's own Type Enforcement plus category matching STILL requires the accessing process's MCS categories to match the target resource's MCS categories — a genuinely separate security mechanism that doesn't simply rely on the same namespace boundary that was already potentially compromised, which is the core "defense in depth" value sVirt adds beyond namespaces/cgroups alone.

**2. Two containers on the same host both run under the same base SELinux type (e.g., `container_t`). Does that mean SELinux treats them identically from a security perspective? Why or why not?**

No — while they DO share the same base type (meaning the general Type Enforcement rules governing what "container_t processes" are broadly allowed to do apply equally to both), each container is typically assigned its OWN distinct, randomly-generated MCS category set as part of its context's level field. Since SELinux's MCS enforcement requires a category match for access (not just a type match), the two containers are still meaningfully isolated FROM EACH OTHER despite sharing the same type — Container A's process, even though it's the same `container_t` type as Container B, cannot access Container B's `container_file_t`-labeled files because their respective category sets don't match, which is a genuinely different, additional dimension of isolation beyond what type alone would provide.

**3. If you needed to grant two SPECIFIC containers the ability to share a particular volume/resource with each other, while keeping them isolated from every OTHER container on the host, how would you approach this using what you know about MCS categories?**

I'd assign both of those specific containers OVERLAPPING or matching MCS categories for the specific shared resource (or, depending on the container runtime's tooling, explicitly configure them to use the SAME category set for that particular shared volume specifically), while leaving their otherwise-distinct, randomly-assigned categories in place for everything else — this lets exactly those two containers' category sets match for the shared resource specifically, permitting the intended sharing, while every OTHER container on the host, with its own different random categories, remains unable to access that same resource, since its categories wouldn't match. The specific mechanics depend on the container runtime/orchestration tooling in use (Podman, Docker, Kubernetes each have their own way of exposing this), but the underlying SELinux principle — matching categories enable access, mismatched categories block it, even under an otherwise-identical type — is what's actually being leveraged either way.

**4. Why is understanding sVirt/MCS categories specifically relevant for someone whose Linux SysAdmin career is heading toward SRE/DevOps/cloud-adjacent roles, beyond general SELinux knowledge?**

As infrastructure increasingly shifts toward container-based and orchestrated (Kubernetes-style) deployment models, understanding the SECURITY ISOLATION MECHANISMS specific to that world — not just "how do I run a container" but "what's actually preventing one container from accessing another's data even under a shared host kernel" — becomes directly relevant to designing, troubleshooting, and explaining the security posture of container-based infrastructure, which is exactly the kind of knowledge a product-company interviewer assessing someone's readiness for a more cloud/container-oriented role would want to confirm goes beyond a surface "containers use namespaces" answer. It's also directly relevant for troubleshooting: an unexpected "permission denied" inside a container that looks like it SHOULD have access, based on standard Unix permissions and even the container's basic type, might actually be an MCS category mismatch — exactly the kind of subtle, SELinux-specific container issue that a purely container-runtime-focused (rather than underlying-OS-focused) engineer might not think to check.

**5. A container is failing to access a mounted volume with a "permission denied" error, despite the volume's standard Unix permissions being correctly configured for the container's UID. What SELinux-specific thing would you check, given everything covered today, before assuming it's a namespace/UID mapping issue?**

I'd check the ACTUAL SELinux context of the volume's files on the host (`ls -Z` on the underlying host path) and compare it against the container process's own SELinux context/MCS categories (`podman inspect <container> --format '{{.ProcessLabel}}'` or the equivalent for the runtime in use) — a mismatch between the volume's labeled type/category and what the container's process is permitted to access, exactly as demonstrated in this topic's lab, would produce precisely this "permission denied despite correct Unix permissions" symptom, and is a genuinely common cause of container volume-mount access issues that's easy to misdiagnose as a UID/namespace mapping problem if you're not specifically thinking to check the SELinux layer. The fix, depending on the exact mismatch found, might involve relabeling the volume with an appropriate type/category (many container runtimes offer a `:z` or `:Z` mount option specifically for this — shared vs. private relabeling of bind-mounted volumes) rather than anything related to UID mapping at all.

---

**End of Day 7.** You should now be able to use permissive mode correctly as a diagnostic tool without treating it as a fix, reason precisely about Type Enforcement as two independent, sequentially-checked access control systems, distinguish boolean-shaped problems from context-shaped ones and fix each with the right tool, work the complete denial-troubleshooting decision tree in the correct order with `setenforce 0` never as a terminal step, and — as the advanced capstone — explain how sVirt's MCS category isolation provides container-to-container security beyond what namespaces and cgroups alone deliver, directly relevant to where this guide's later chapters take your career next.

Proceed to **Day 8 — Shared Storage & Time (AutoFS/NFS, Chrony)** next.
