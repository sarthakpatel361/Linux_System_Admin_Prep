# Day 6 — Firewalld & Network Security (Full Study Edition, with Answers)

**Estimated time: 4–5 hours**

Firewalld interview questions at a product company rarely stop at "how do you open a port." They probe whether you understand the *trust model* zones express, whether you can write a rich rule that does exactly what's intended (not almost), and — since firewalld has sat on top of **nftables**, not iptables, since RHEL 8 — whether you know what's actually enforcing your rules underneath the `firewall-cmd` abstraction. That last point is this day's bonus topic, and it directly extends the `nft list ruleset` command Chapter 0 introduced without ever explaining.

---

## Topic 1: Zones — The Trust Model, Not Just Rule Containers

### Quick Review
- A **zone** represents a level of **trust** for a network interface/source, not just an arbitrary bucket of rules.
- Default zone: `public` (untrusted). Also common: `trusted` (allow everything), `internal` (higher trust than public), `drop` (silently discard everything).
- An interface belongs to exactly one zone at a time; a multi-homed server can assign different zones to different NICs.
- `firewall-cmd --get-active-zones` shows what's actually in effect right now — don't assume from config files alone.

### Quick Learning

The conceptual mistake candidates make is treating zones like arbitrary named rule-groups you could organize however you like — the actual design intent is that a zone represents a *trust level*, and you assign interfaces to zones based on how much you trust the network reachable through that interface. This reframing is what lets you answer the real interview question correctly: "why would you assign different NICs on the same server to different zones" — because a management NIC on an internal, controlled network genuinely deserves different default trust than a public-facing NIC, and zones are the mechanism for expressing that difference declaratively rather than writing bespoke rules for each interface from scratch.

**A multi-homed server, two interfaces, two trust levels:**
```
        RHEL Server (web + admin backend)
     ┌───────┴───────┐
   eth0                eth1
  Public internet      Internal mgmt network
  10.0.0.5              10.1.0.5
        │                    │
        ▼                    ▼
  zone: public          zone: internal
  ──────────────         ──────────────
  Only 80/443 open       SSH open, more
  Default-deny            permissive by
  everything else         default (trusted
                           network segment)

  firewall-cmd --zone=public   --change-interface=eth0
  firewall-cmd --zone=internal --change-interface=eth1

  Traffic hitting eth0 is evaluated against PUBLIC zone rules.
  Traffic hitting eth1 is evaluated against INTERNAL zone rules.
  Same server, same firewalld instance, two completely different
  effective policies based on WHICH interface the traffic arrives on.
```

### Implementation (Learn by Applying)

**Scenario:** Configure a dual-homed server correctly — public-facing web traffic on one interface locked down to just 80/443, internal admin access on a second interface with more permissive defaults appropriate for a trusted internal segment.

```bash
firewall-cmd --get-zones
firewall-cmd --get-active-zones
firewall-cmd --get-default-zone

# Assign each interface to the zone matching its actual trust level
firewall-cmd --zone=public --change-interface=eth0 --permanent
firewall-cmd --zone=internal --change-interface=eth1 --permanent

firewall-cmd --zone=public --add-service=http --permanent
firewall-cmd --zone=public --add-service=https --permanent
firewall-cmd --zone=internal --add-service=ssh --permanent
firewall-cmd --zone=internal --add-service=cockpit --permanent   # example internal-only admin tool

firewall-cmd --reload
firewall-cmd --zone=public --list-all
firewall-cmd --zone=internal --list-all
```

Confirm the zone assignment actually took effect at the interface level, not just in config:
```bash
firewall-cmd --get-active-zones
# Expected output pairs each active zone with its actual interface(s):
#   public
#     interfaces: eth0
#   internal
#     interfaces: eth1
```

### Interview Questions — with Answers

**1. Why would a team deliberately assign different zones to different NICs on the same multi-homed server, rather than using one zone with a unified rule set for everything?**

Because the two interfaces genuinely face different trust environments — a public-facing NIC receiving arbitrary internet traffic warrants a restrictive, default-deny posture, while a NIC connected to an internal, access-controlled management network can reasonably have a more permissive baseline without meaningfully increasing risk, since that network segment already has its own access controls. Using one unified zone/rule set for both would force a choice between being needlessly restrictive on the internal-facing traffic (adding friction for legitimate internal admin work) or needlessly permissive on the public-facing traffic (a real security exposure) — assigning zones per-interface lets each interface's policy match its actual trust level precisely, which is the entire design purpose zones exist to serve.

**2. What's the practical difference between the `drop` zone and simply not opening any ports in the `public` zone — don't both end up blocking all traffic?**

The `public` zone with no ports opened still allows certain baseline behaviors by default depending on configuration (e.g., it may still respond to some ICMP, and critically, it still processes and can send back explicit REJECT responses for blocked traffic rather than silently discarding it, depending on default rule behavior) — it's a "default-deny but present, and configurable" zone. The `drop` zone is more absolute: it's specifically designed to silently discard essentially all incoming traffic with no response at all, not even a rejection — from the perspective of anyone scanning or probing the host, a `drop`-zoned interface can appear as if nothing is there at all, versus a restrictively-configured `public` zone which, while blocking access, may still reveal the host's presence through its handling of blocked connection attempts. `drop` is the appropriate choice for scenarios needing maximum stealth/hardening (e.g., an interface facing a genuinely hostile network segment), while `public` with minimal services opened is the more common, standard default-deny posture for typical internet-facing servers.

**3. A server has two active zones assigned to two different interfaces, but traffic that should be allowed on one interface is being blocked. What would you check first?**

First, `firewall-cmd --get-active-zones` to confirm which zone is ACTUALLY bound to which interface right now — it's possible the zone assignment itself doesn't match what's assumed (e.g., a NIC renamed after a reboot, or a config change that was never actually reloaded/applied). Then I'd check `firewall-cmd --zone=<the relevant zone> --list-all` to confirm the specific service/port genuinely is allowed in THAT zone specifically — a common mistake is opening a port in the wrong zone (e.g., opening it in `public` when the traffic is actually arriving via the interface bound to `internal`), which looks correct in isolation but doesn't apply to the actual traffic path involved.

**4. What happens to a zone's changes made with `firewall-cmd --zone=public --add-service=http` (no `--permanent` flag) after the next `firewall-cmd --reload` or a system reboot?**

Without `--permanent`, the change is applied only to the RUNTIME configuration — it takes effect immediately but is not written to the persistent configuration files at all. A `--reload` specifically re-reads the persistent configuration and reapplies it as the new runtime state, which means a non-permanent change made earlier gets WIPED OUT the moment `--reload` runs, since reload has no knowledge of it — it never existed anywhere except transient runtime state. Similarly, a reboot starts firewalld fresh from the persistent configuration only, so any change made without `--permanent` is gone after a reboot as well. This is a very common practical mistake: testing a change live without `--permanent` (reasonable for testing), forgetting to make it permanent once confirmed correct, and then losing the change silently at the next reload/reboot with no error or warning that this happened.

**5. How would you audit a running production server to confirm the difference (if any) between its RUNTIME firewall state and its PERMANENT (saved) configuration — why might these two ever diverge in the first place?**

`firewall-cmd --zone=<zone> --list-all` (runtime, the default) versus `firewall-cmd --zone=<zone> --list-all --permanent` shows the two states side by side for direct comparison — any discrepancy between them reveals exactly what's been changed live but not yet persisted (or, less commonly, persisted changes not yet applied to the current runtime state, which would require a reload to reconcile). These two states diverge specifically because of the scenario in question 4 — someone made a live change to test something and either intentionally left it non-permanent (a deliberate, temporary test) or forgot to add `--permanent` when they meant the change to be lasting — auditing for this divergence is exactly how you catch a server that LOOKS correctly configured right now but will silently revert to a different, potentially less secure state at the next reload or reboot.

---

## Topic 2: Rich Rules — Precise, Conditional Firewall Logic

### Quick Review
- Rich rules let you express conditions beyond simple "open this port" — source/destination IP, logging, rate limiting, and explicit accept/reject/drop actions combined in one rule.
- Rich rules are evaluated with **higher priority** than the zone's basic port/service rules by default.
- Rule order among MULTIPLE rich rules in the same zone matters — more specific rules should generally be listed/considered before broader ones, though firewalld's internal priority handling has its own nuances worth verifying with `--list-rich-rules`.
- A rich rule with `log` lets you audit specific traffic patterns without needing full packet capture.

### Quick Learning

Basic `--add-service`/`--add-port` commands answer "is this port open in this zone," a yes/no question with no room for nuance. Rich rules exist for exactly the cases where the real requirement is conditional: "allow SSH, but only from this specific subnet," or "allow this port from most sources, but explicitly reject it from this one known-bad range while logging the attempt." The interview-level skill here isn't just rich-rule syntax — it's recognizing when a requirement genuinely needs a rich rule versus when a simpler port/service rule would suffice, since reaching for rich-rule complexity when it isn't needed makes a firewall configuration harder to audit later.

**Simple rule vs. rich rule — same port, fundamentally different precision:**
```
  Simple:  firewall-cmd --add-port=3306/tcp
           ─────────────────────────────────
           Port 3306 open to EVERY source reaching this zone.
           No way to express "only from the app servers."

  Rich:    firewall-cmd --add-rich-rule=
             'rule family="ipv4" source address="10.0.5.0/24"
              port port="3306" protocol="tcp" accept'
           ─────────────────────────────────────────────────
           Port 3306 open ONLY to sources in 10.0.5.0/24.
           Traffic from anywhere else hitting 3306 falls
           through to the zone's default policy (typically deny).

  Combined with an explicit deny + logging for a known-bad range:
  firewall-cmd --add-rich-rule=
    'rule family="ipv4" source address="203.0.113.0/24"
     port port="3306" protocol="tcp"
     log prefix="db-blocked" level="warning" limit value="3/m" reject'
           ─────────────────────────────────────────────────
           Explicitly REJECTS (not just falls through to default)
           traffic from this specific range, LOGS each attempt
           (rate-limited to 3/minute to avoid log flooding), giving
           you both the block AND the audit trail in one rule.
```

### Implementation (Learn by Applying)

**Scenario:** A database port (3306) needs to be reachable only from the application tier's subnet, with all other traffic explicitly logged and rejected (not silently dropped) so you have an audit trail of anyone probing it.

```bash
firewall-cmd --zone=internal --add-rich-rule='rule family="ipv4" source address="10.0.5.0/24" port port="3306" protocol="tcp" accept' --permanent

firewall-cmd --zone=internal --add-rich-rule='rule family="ipv4" port port="3306" protocol="tcp" log prefix="db-unauthorized" level="warning" limit value="5/m" reject' --permanent

firewall-cmd --reload
firewall-cmd --zone=internal --list-rich-rules
```

Test both paths and verify the logging:
```bash
# From an app-tier host (10.0.5.x) — should succeed
nc -zv db-server 3306

# From anywhere else — should be explicitly rejected, not just time out
nc -zv db-server 3306    # (run from a host outside 10.0.5.0/24)

journalctl -k | grep "db-unauthorized"    # Confirm the rejected attempt was actually logged
```

### Interview Questions — with Answers

**1. Why might you deliberately choose `reject` over the zone's simpler default-deny behavior for unauthorized traffic to a sensitive port, rather than just leaving it unlisted and letting it fall through?**

Letting traffic fall through to the zone's default (typically a silent drop with no explicit rule) provides no logging or audit trail by default — you'd have no record of who attempted to reach that port. An explicit rich rule with `reject` (optionally combined with `log`, as in the lab) gives you both an immediate, explicit response to the connecting client AND a deliberate, rate-limited audit log entry for every such attempt — turning "nobody's probing our database port, presumably" into "here's an actual log showing exactly who tried and when," which matters significantly for security monitoring and incident investigation.

**2. What's the difference between `reject` and `drop` as rich-rule actions, and when would you deliberately choose one over the other for a security-sensitive rule?**

`reject` sends back an explicit response (a TCP RST or ICMP unreachable, depending on protocol) informing the client the connection was actively refused — fast, clear feedback, but it also confirms to a potential attacker/scanner that something is actually listening/reachable at that address, just refusing this specific traffic. `drop` silently discards the packet with zero response, leaving the connecting party to eventually time out with no information about whether anything is even there at all. For a genuinely sensitive, security-critical service, I'd lean toward `drop` for unauthorized sources specifically, since it denies a potential attacker even the confirmation that a service exists at that port — at the cost of legitimate misconfigured clients getting a less immediately diagnosable "hangs and times out" experience rather than a clear rejection, which is a real, deliberate tradeoff between security posture and troubleshooting friendliness.

**3. You've added a rich rule allowing traffic from a specific subnet, but traffic from that subnet is STILL being blocked. What are the possible causes, given what you know about zones and rule precedence?**

First, I'd confirm the rich rule was actually applied to the correct ZONE — the one actually bound to the interface the traffic arrives through, per Topic 1's lessons; a rich rule added to the wrong zone has zero effect on traffic evaluated against a different zone. Second, I'd check for another, more specific or conflicting rich rule that might be taking precedence and explicitly rejecting/dropping the same traffic before the intended allow rule gets a chance to apply — `firewall-cmd --zone=<zone> --list-rich-rules` shows the full set, and I'd review them for any overlapping or contradictory entries. Third, I'd verify the rule was actually made `--permanent` AND that `--reload` was run (or the change was also applied live) — a permanent-only change without a reload, or a runtime-only change that got reloaded away, are both scenarios from Topic 1 that would produce exactly this "I added the rule but it's not working" symptom.

**4. What does the `limit value="5/m"` clause in a rich rule's `log` action actually do, and why is it important to include when logging potentially high-frequency events like blocked connection attempts?**

It rate-limits how often this specific logging action fires — in this example, at most 5 log entries per minute for matches against this rule, regardless of how many actual packets match. This matters because an actual attack or aggressive scanning activity against a blocked port could generate an enormous volume of matching packets in a short time — without a rate limit, logging EVERY single one could flood the system log with thousands of near-identical entries, consuming disk space, making the log harder to actually review usefully, and in extreme cases contributing to a denial-of-service condition against the logging subsystem itself. The rate limit preserves the VALUE of the log (you still get confirmation that this is happening and roughly how often) without the volume becoming a problem in its own right.

**5. Explain the actual precedence relationship between rich rules and the zone's basic port/service rules — if a rich rule and a basic `--add-port` entry seem to conflict, which one actually wins, and why does understanding this matter for writing correct firewall policy?**

Rich rules are evaluated with higher priority than the zone's basic port/service entries by default — meaning a rich rule's explicit accept/reject/drop for a given match takes precedence over what the basic service/port list alone would otherwise allow. This matters because it's entirely possible to have a port opened via a simple `--add-port` entry (suggesting it should be broadly accessible) while ALSO having a rich rule that explicitly rejects that same port for certain sources — the rich rule's more specific, higher-priority action wins for traffic matching its conditions, while traffic NOT matching the rich rule's specific conditions still falls back to the basic port rule's broader allowance. Understanding this precedence is what lets you correctly build the common "generally open, but explicitly restricted for specific cases" pattern seen in the lab — get the precedence backwards in your mental model, and you'll mispredict which rule actually governs a given piece of traffic.

---

## Topic 3: Service and Port Management — Predefined Services vs. Custom Ports

### Quick Review
- `--add-service=http` uses a predefined, named service definition (port + protocol, sometimes more) maintained by firewalld — more readable and self-documenting than raw port numbers.
- `--add-port=8080/tcp` opens a specific port/protocol combination directly, for anything without (or not needing) a predefined service definition.
- You can define your own custom service XML for a recurring, organization-specific port/protocol combination, making firewall policy more maintainable at scale.
- `firewall-cmd --get-services` lists everything firewalld already knows as a predefined service — check before reinventing with raw ports.

### Quick Learning

Using named services instead of raw port numbers where possible is a real operational best practice, not just a style preference — a rule reading `--add-service=https` is immediately self-documenting to the next admin who reads the configuration, whereas `--add-port=443/tcp` requires them to already know what commonly runs on 443. For genuinely custom, organization-specific services, defining your own named service (rather than always using raw ports) extends that same self-documentation benefit to internal applications too, and makes bulk firewall audits and policy reviews significantly easier to reason about at a glance.

**Predefined service vs. raw port — the same open port, different clarity:**
```
  firewall-cmd --zone=public --list-all
  ──────────────────────────────────────
  services: ssh https                    <- immediately clear what these are for
  ports: 8443/tcp                        <- what IS this? Reader has to go find out
                                             elsewhere, or ask someone, or guess

  Defining a custom service instead:
  /etc/firewalld/services/myapp-api.xml
     <?xml version="1.0" encoding="utf-8"?>
     <service>
       <short>MyApp API</short>
       <description>Internal REST API for MyApp platform</description>
       <port protocol="tcp" port="8443"/>
     </service>

  firewall-cmd --zone=public --add-service=myapp-api --permanent
  firewall-cmd --zone=public --list-all
  ──────────────────────────────────────
  services: ssh https myapp-api          <- now self-documenting, same as the
                                             built-in services, for YOUR org's
                                             own internal application
```

### Implementation (Learn by Applying)

**Scenario:** An internal application listens on port 8443 for its REST API. Rather than leaving it as an opaque raw port in the firewall configuration, define it as a proper named service so it's self-documenting for every future admin who reviews this server's configuration.

```bash
firewall-cmd --get-services | tr ' ' '\n' | grep -i http    # Check what's already predefined before reinventing

mkdir -p /etc/firewalld/services
cat > /etc/firewalld/services/myapp-api.xml << 'EOF'
<?xml version="1.0" encoding="utf-8"?>
<service>
  <short>MyApp API</short>
  <description>Internal REST API for the MyApp platform, used by the app tier</description>
  <port protocol="tcp" port="8443"/>
</service>
EOF

firewall-cmd --reload                       # Required to pick up the new service definition file
firewall-cmd --get-services | grep myapp

firewall-cmd --zone=public --add-service=myapp-api --permanent
firewall-cmd --reload
firewall-cmd --zone=public --list-all       # Now shows "myapp-api" instead of an opaque raw port
```

Compare against the raw-port approach directly, for contrast:
```bash
firewall-cmd --zone=public --add-port=9090/tcp --permanent    # A quick, one-off port with no custom service defined
firewall-cmd --reload
firewall-cmd --zone=public --list-all       # Shows "9090/tcp" in the ports: line — functional, but not self-documenting
```

### Interview Questions — with Answers

**1. What's the practical, operational benefit of defining a custom firewalld service for an internal application, rather than just always using `--add-port` with the raw port number?**

A raw port number in firewall configuration output tells the next person reviewing it nothing about WHAT that port is for or WHY it's open — they'd need external documentation, tribal knowledge, or direct investigation to understand it. A custom service definition with a clear `<short>` name and `<description>` makes the firewall configuration self-documenting directly within `firewall-cmd --list-all` output — someone auditing the server's open ports sees "myapp-api" and immediately understands its purpose, the same way they'd understand a built-in "https" entry, without needing to consult separate documentation or ask around to figure out what an opaque port number represents.

**2. If an internal application's port ever changes (say, moving from 8443 to 8444), what's the operational advantage of having defined it as a custom service versus having used a raw `--add-port` rule directly?**

With a custom service definition, the port number lives in exactly ONE place — the service's XML definition file — and updating it there automatically applies everywhere that service is referenced across every zone it's been added to, without needing to hunt down and update every individual `--add-port` rule across potentially multiple zones/servers where the raw port number might have been used directly. With raw `--add-port` rules, the port number is duplicated wherever it was configured, and a port change requires finding and updating every one of those individual instances — the custom-service approach centralizes the definition, making this kind of change both easier and less error-prone (less risk of missing an instance during the update).

**3. You define a new custom service XML file, but `firewall-cmd --get-services` still doesn't show it. What did you likely forget?**

Most likely, a `firewall-cmd --reload` was never run after creating the new service XML file — firewalld reads service definitions from disk at startup/reload time, not continuously watching the filesystem for new files, so a newly created service definition file won't be recognized until firewalld is explicitly told to re-read its configuration. I'd also double check the file was placed in the correct directory (`/etc/firewalld/services/` for a system-wide custom service) and that the XML itself is well-formed (a malformed XML file may simply be silently ignored/skipped by firewalld rather than producing an obvious error).

**4. What's the difference between a service defined in `/usr/lib/firewalld/services/` versus one defined in `/etc/firewalld/services/`, and which should you use for a custom, organization-specific service?**

`/usr/lib/firewalld/services/` holds the package-provided, vendor-shipped default service definitions (http, https, ssh, and so on) — these are managed by the firewalld package itself and can be overwritten during package updates, exactly analogous to the vendor-unit-file situation with systemd from Day 3. `/etc/firewalld/services/` is where custom, admin-defined service files belong (as used in the lab) — this location is never touched by package updates, so a custom organization-specific service definition placed here persists reliably across firewalld package upgrades, the same reasoning that makes `systemctl edit` drop-ins the correct pattern for customizing systemd units rather than editing vendor files directly.

**5. How would you audit a server to find every currently-open port that ISN'T accounted for by a clearly-named service — i.e., find the "mystery" raw ports that need investigation or proper documentation?**

`firewall-cmd --zone=<zone> --list-all` (run across every active zone) directly separates its output into a `services:` line (named, self-documenting entries) and a `ports:` line (raw port/protocol pairs with no named service backing them) — any entry appearing in the `ports:` line specifically represents exactly this category of "opaque, needs investigation or proper documentation" configuration. I'd treat every raw port entry found this way as a follow-up task: identify what's actually using it (`ss -tulnp` cross-referenced against the port number), confirm it's still genuinely needed, and either define a proper custom service for it (if it's a legitimate, ongoing need) or remove the rule entirely if it turns out to be stale/no-longer-needed — raw ports accumulating over time with no documentation is a realistic, common source of firewall configuration drift that periodic review like this is meant to catch.

---

## Topic 4: Masquerading and NAT — When This Server IS the Gateway

### Quick Review
- **Masquerading** (`--add-masquerade`) enables NAT, translating outbound traffic from other hosts routed THROUGH this server to appear as if it originated from this server's own IP.
- Required whenever a RHEL server acts as a router/gateway for other hosts (e.g., a bastion/NAT gateway, a container host routing container traffic out).
- Masquerading alone doesn't grant connectivity — `net.ipv4.ip_forward` (a kernel sysctl) must also be enabled for the box to actually forward packets between interfaces at all.
- Masquerading is zone-scoped, same as any other firewalld setting — it applies to the zone bound to the interface traffic is masqueraded OUT through.

### Quick Learning

Masquerading answers a genuinely different question than every other firewalld topic covered today — those are all about "what traffic is allowed to reach THIS server," while masquerading is about "this server is itself acting as a network intermediary for OTHER hosts' traffic passing through it." The most common real interview trap: someone enables masquerading, expects traffic to now route/NAT correctly, and it still doesn't work — because masquerading is a firewalld/NAT-layer setting, but actual packet FORWARDING between interfaces is a separate kernel-level setting (`net.ipv4.ip_forward`) that has to be enabled independently; forgetting one while configuring the other is a very common, easy-to-make mistake.

**Two separate switches that BOTH need to be on for a NAT gateway to actually work:**
```
        Internal hosts (10.0.2.0/24)
                    │
                    ▼
              RHEL Server acting as gateway
           ┌─────────────────────────────┐
           │  eth1 (internal, 10.0.2.1)     │
           │        │                       │
           │  Switch #1:                    │
           │  net.ipv4.ip_forward = 1        │  <- KERNEL setting: "am I even
           │  (sysctl, NOT a firewalld        │     ALLOWED to forward packets
           │   concept at all)                │     between my own interfaces
           │        │                       │     at all?"
           │        ▼                       │
           │  eth0 (public, 203.0.113.5)    │
           │        │                       │
           │  Switch #2:                    │
           │  firewall-cmd --add-masquerade  │  <- FIREWALLD setting: "when I
           │  (on the zone bound to eth0)    │     DO forward this traffic out,
           └─────────────────────────────┘     rewrite its source IP to look
                    │                             like it came from ME"
                    ▼
              Public internet

  BOTH switches must be ON. ip_forward alone routes packets but with
  their ORIGINAL internal source IP (likely unusable/unroutable on
  the public internet, and exposes internal addressing). Masquerade
  alone means nothing if ip_forward is off, since packets never
  even get forwarded between interfaces in the first place.
```

### Implementation (Learn by Applying)

**Scenario:** Configure a RHEL server to act as a NAT gateway for an internal subnet — reproduce the common "I set up masquerading but internal hosts still can't reach the internet" mistake by forgetting `ip_forward` first, diagnose it, then fix it completely.

```bash
# Confirm current forwarding state - very likely OFF by default
sysctl net.ipv4.ip_forward

# Enable masquerading FIRST, deliberately, to reproduce the incomplete-fix scenario
firewall-cmd --zone=public --add-masquerade --permanent
firewall-cmd --reload
firewall-cmd --zone=public --query-masquerade

# From an internal host routed through this gateway, test connectivity — still fails
# (assume 10.0.2.1 is this server's internal-facing IP, internal host has it as its gateway)
ping -c2 -I eth1 8.8.8.8     # Fails, or packets simply never leave this server at all
```

Diagnose and complete the fix:
```bash
sysctl net.ipv4.ip_forward     # Confirm it's still 0 — this is the missing piece

# Enable forwarding, both immediately and persistently
sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.d/99-nat-gateway.conf
sysctl -p /etc/sysctl.d/99-nat-gateway.conf

# NOW test again from the internal host's perspective
ping -c2 8.8.8.8              # Should succeed now that BOTH switches are on

# Confirm what's actually happening at the packet level
firewall-cmd --zone=public --list-all | grep masquerade
conntrack -L 2>/dev/null | grep -c ESTABLISHED    # Active NAT'd connections, if conntrack-tools installed
```

### Interview Questions — with Answers

**1. You've enabled `firewall-cmd --add-masquerade` on the correct zone, but internal hosts still can't reach the internet through this server. What's the most likely missing piece, and why does it exist as a separate setting from masquerading in the first place?**

The most likely missing piece is `net.ipv4.ip_forward` still being disabled at the kernel level — masquerading only controls how traffic gets NAT'd/rewritten once it's actually being forwarded, but the kernel's ability to forward packets between network interfaces AT ALL is a separate, more fundamental setting that has to be explicitly enabled. These exist as separate settings because they serve genuinely different purposes at different layers: `ip_forward` is a basic kernel routing capability question (should this host route traffic between interfaces at all, a meaningful security-relevant default to leave OFF on non-gateway servers), while masquerading is specifically a NAT/firewall-layer concern about HOW forwarded traffic's source address gets rewritten — a server could theoretically forward traffic without masquerading it (routing internal, routable addresses without NAT) which is a legitimately different configuration from the masquerade-NAT gateway pattern, so conflating the two into one setting would remove that flexibility.

**2. Why would a server administrator deliberately leave `net.ipv4.ip_forward` disabled on most servers, even ones with multiple network interfaces?**

Most multi-homed servers (e.g., the dual-zone web server from Topic 1) have multiple interfaces for trust-segmentation or redundancy reasons, NOT because they're meant to act as a routing intermediary passing OTHER hosts' traffic between those interfaces — leaving `ip_forward` disabled by default ensures such a server can't accidentally become an unintended routing path between two network segments it's connected to, which could bypass intended network segmentation/security boundaries if it were enabled without a specific, deliberate reason. It's specifically enabled only on servers genuinely intended to function as routers/gateways, as a deliberate, explicit choice — not a default assumption for every multi-NIC server.

**3. What's the difference between routing traffic between two internal, routable subnets versus masquerading traffic destined for the public internet — why isn't masquerading always necessary just because forwarding is happening?**

If both the source and destination networks use addressing that's genuinely routable and reachable end-to-end without translation (for example, routing between two internal corporate subnets that both have proper routing table entries pointing back appropriately), the traffic can simply be forwarded as-is, with its original source address intact, and return traffic can find its way back correctly through normal routing — no NAT/masquerading needed. Masquerading becomes necessary specifically when the traffic is crossing a boundary where the original source address either isn't routable on the destination network (private RFC1918 addresses reaching the public internet, which doesn't know how to route back to a private address) or where you deliberately want to hide/obscure the internal addressing structure from the destination network for security reasons — forwarding and masquerading are answering different questions ("can this traffic get there at all" versus "should the traffic's origin be rewritten"), and only some forwarding scenarios also require masquerading.

**4. How would you verify, at the packet level, that masquerading is actually rewriting source addresses correctly, rather than just trusting that `firewall-cmd --query-masquerade` reports it as enabled?**

I'd use `tcpdump` simultaneously on both the internal-facing and external-facing interfaces of the gateway server while generating test traffic from an internal host — comparing the source IP address visible in the packet capture on the EXTERNAL interface against the original internal host's actual IP visible on the INTERNAL interface capture confirms whether the rewrite is genuinely happening: the external capture should show the gateway's OWN public IP as the source, not the internal host's private IP, if masquerading is functioning correctly. Additionally, `conntrack -L` (if `conntrack-tools` is installed) shows the actual NAT connection tracking table, directly revealing the original-to-translated address mappings firewalld/netfilter is maintaining for active connections, which is a more direct confirmation than relying solely on the configuration reporting a setting as "enabled" without verifying its actual runtime effect.

**5. A server needs to route traffic between two INTERNAL, non-public subnets (no internet-facing NAT involved at all) — does it need masquerading enabled, forwarding enabled, both, or neither?**

It needs `ip_forward` enabled, since the server is genuinely acting as a routing intermediary between the two subnets and needs the fundamental kernel capability to forward packets between its interfaces. It does NOT necessarily need masquerading — if both internal subnets use properly routable (even if private/RFC1918) addressing with correct routing table entries on both sides pointing back through this gateway appropriately, traffic can be forwarded with its original source addresses intact and return traffic will correctly route back without any NAT/address-rewriting needed at all; masquerading would only become necessary here if, for some specific reason, you wanted to obscure one subnet's addressing from the other or if proper bidirectional routing between the two subnets genuinely couldn't be established without it.

---

## Topic 5 (Bonus — Advanced): The nftables Backend — What's Actually Enforcing Your Rules

### Quick Review
- Since RHEL 8, firewalld's default backend is **nftables**, not the older iptables/xtables framework.
- `firewall-cmd` is a high-level abstraction — every command ultimately translates into actual nftables rules under the hood.
- `nft list ruleset` shows the REAL, low-level rules currently enforced by the kernel — the ground truth beneath `firewall-cmd`'s abstraction.
- **Panic mode** (`firewall-cmd --panic-on`) is firewalld's emergency kill-switch — instantly blocks ALL traffic, both directions, useful during an active incident.

### Quick Learning

Everything covered today has been at the `firewall-cmd`/firewalld abstraction layer — genuinely useful and the correct day-to-day interface, but it's still an abstraction over something more fundamental: the kernel's actual packet-filtering engine, which on modern RHEL is **nftables**. Interviewers ask about this specifically to test whether you understand you're working with an abstraction at all, and whether you know how to drop down to the real, underlying ruleset when `firewall-cmd`'s view of the world doesn't fully explain observed behavior — the same "know the abstraction AND what's beneath it" pattern that showed up with Stratis-over-device-mapper on Day 1.

**The actual layering — firewall-cmd is a translator, not the enforcement engine itself:**
```
  You run:
     firewall-cmd --zone=public --add-service=https --permanent
              │
              ▼
     firewalld daemon translates this into actual nftables rule
     syntax and programs it into the kernel's nftables tables
              │
              ▼
     nft list ruleset          <- THIS is the real, ground-truth
                                   view of what's actually being
                                   enforced by the kernel right now —
                                   firewall-cmd's own "--list-all"
                                   output is firewalld's OWN
                                   interpretation/summary, one layer
                                   removed from the literal kernel state

  When firewall-cmd's reported state and observed real traffic
  behavior seem to disagree, `nft list ruleset` is where you go
  to see what's ACTUALLY being enforced, bypassing firewalld's
  abstraction entirely.
```

### Implementation (Learn by Applying)

**Scenario:** You've configured firewalld rules throughout today's labs. Drop down to the nftables layer to see exactly what firewalld actually programmed into the kernel — then use panic mode to understand firewalld's emergency response capability, and understand why mixing manual nftables rules with firewalld is generally discouraged.

```bash
# See the REAL rules currently enforced, beneath the firewall-cmd abstraction
nft list ruleset | less

# Find firewalld's own tables specifically within the broader nftables output
nft list tables
nft list table inet firewalld     # firewalld's actual programmed rules, in native nftables syntax
```

Compare firewalld's summary view against the raw nftables reality for the rich rule from Topic 2:
```bash
firewall-cmd --zone=internal --list-rich-rules
nft list table inet firewalld | grep -A3 3306    # Find the corresponding raw nftables rule this translated into
```

Practice panic mode — firewalld's emergency, instant, total lockdown:
```bash
firewall-cmd --panic-on              # Immediately blocks ALL traffic, both directions, bypassing all normal rules
firewall-cmd --query-panic           # Confirm it's active
# CAUTION: this will drop your own SSH session's ability to pass NEW traffic — understand this before running it live
ping -c2 8.8.8.8 2>&1                # Fails completely, even things normally allowed

firewall-cmd --panic-off             # Restore normal rule enforcement
firewall-cmd --query-panic
```

Understand (without necessarily doing, given the risk) why direct manual nftables edits alongside firewalld are discouraged:
```bash
firewall-cmd --direct --get-all-rules 2>/dev/null   # firewalld's own (deprecated in favor of full nftables passthrough
                                                        in newer versions, but illustrative) mechanism for injecting
                                                        custom rules WITHOUT bypassing firewalld's own management
```
Directly running raw `nft add rule ...` commands outside of firewalld's own management creates rules firewalld has no knowledge of and won't preserve, reconcile, or display in its own `--list-all` output — the next `firewall-cmd --reload` can wipe out manually-added nftables rules that firewalld didn't create and doesn't track, since reload reprograms the ruleset purely from firewalld's own persistent configuration, with no awareness of anything added outside that mechanism.

### Interview Questions — with Answers

**1. Why is it important to know that firewalld's backend on modern RHEL is nftables, not iptables — what practical difference does this make for a working SysAdmin, beyond trivia?**

Beyond simply knowing the correct terminology, this matters because troubleshooting sometimes requires dropping below the `firewall-cmd` abstraction to see the actual, ground-truth rules the kernel is enforcing — and using the correct tool for that (`nft list ruleset`) rather than an outdated one (`iptables -L`, which on a modern RHEL system with the nftables backend may show an empty or misleadingly incomplete picture, since the real rules live in nftables tables, not the legacy iptables rule format). Someone still reflexively reaching for `iptables -L` to troubleshoot a firewalld issue on RHEL 8/9 is troubleshooting with an outdated mental model of the system that can lead to incorrect conclusions (e.g., "there are no firewall rules at all" when there genuinely are, just not visible through that particular legacy lens).

**2. `firewall-cmd --zone=public --list-all` shows a specific rich rule as active, but you suspect the actual traffic behavior doesn't match what that rule should be doing. How would you confirm what's genuinely being enforced, bypassing firewalld's own reporting?**

I'd go directly to `nft list ruleset` (or more specifically `nft list table inet firewalld` to focus on firewalld's own programmed tables specifically) to see the actual, literal nftables rules the kernel is currently enforcing, rather than relying on firewalld's own summary/translation of what it believes it configured. If there's a genuine discrepancy between what `firewall-cmd --list-all` reports and what `nft list ruleset` actually shows being enforced, that's a strong signal something has gone wrong in firewalld's own state management (or, less commonly, something outside firewalld has modified the nftables ruleset directly, per question 4) — going straight to the kernel-level ground truth resolves any ambiguity about what firewalld's abstraction layer is merely reporting versus what's actually happening.

**3. Explain what `firewall-cmd --panic-on` does, and describe a genuine incident-response scenario where you'd actually want to use it, along with the real cost of doing so.**

Panic mode immediately and completely blocks ALL network traffic in both directions, overriding every normal firewalld rule/zone configuration entirely — it's an emergency kill-switch, not a normal operational tool. A genuine scenario: you've discovered active, ongoing data exfiltration or an in-progress compromise on a server and need to IMMEDIATELY sever all its network connectivity to stop further damage while investigation proceeds, faster than carefully crafting and testing specific blocking rules would allow in the moment. The real cost: this also blocks YOUR OWN legitimate access to the server (including your current SSH session's ability to pass new traffic) — meaning you'd need console/out-of-band access (IPMI, hypervisor console) to actually interact with the server further once panic mode is engaged remotely, since panic mode makes no exception for the administrator's own current connection.

**4. Why is manually running raw `nft add rule ...` commands directly, alongside using firewalld for everything else, generally discouraged as a practice?**

Rules added directly via raw `nft` commands outside of firewalld's own management aren't tracked in firewalld's persistent configuration at all — firewalld has no awareness they exist, won't display them in its own `--list-all`/`--list-rich-rules` output, and critically, the next `firewall-cmd --reload` reprograms the entire ruleset purely from firewalld's own saved configuration, which can wipe out those manually-added rules entirely without warning, since reload has no way of knowing they were ever supposed to persist. This creates exactly the kind of "it worked yesterday, mysteriously stopped working today" confusion that's hard to diagnose unless you specifically know to suspect an untracked manual rule that got silently erased by a routine reload — sticking to firewalld's own management interface (including its direct-rule/passthrough mechanisms for genuinely custom needs) keeps the entire ruleset consistently tracked and reload-safe.

**5. If you needed to add a genuinely custom nftables rule that firewalld's normal `firewall-cmd` interface doesn't have a clean way to express, what's the RIGHT way to do that while still keeping it managed and reload-safe, rather than just running raw `nft` commands directly?**

firewalld provides its own mechanisms specifically for this purpose — historically the `--direct` interface (`firewall-cmd --direct --add-rule ...`), and in more recent firewalld versions, a more complete nftables passthrough/custom-rule mechanism that still gets tracked within firewalld's own persistent configuration and survives reloads correctly, unlike raw manual `nft` commands run entirely outside firewalld's awareness. The key distinction is using firewalld's OWN supported mechanism for injecting custom, lower-level rules (whichever specific interface the RHEL version in use provides) rather than bypassing firewalld's management layer entirely — this keeps the custom rule properly tracked, visible in firewalld's own tooling, and safe across reloads, while still achieving whatever specific, non-standard rule behavior the simpler `--add-service`/`--add-port`/`--add-rich-rule` commands couldn't express on their own.

---

**End of Day 6.** You should now be able to design zone assignments around actual trust levels rather than arbitrary grouping, write rich rules that express genuinely conditional logic (including the reject-vs-drop and rate-limited-logging tradeoffs), choose between named services and raw ports with the operational tradeoffs in mind, correctly configure a NAT gateway including the easy-to-forget `ip_forward` piece, and — as the advanced capstone — drop below firewalld's abstraction to the real nftables ruleset when troubleshooting demands ground truth rather than firewalld's own summary of itself.

Proceed to **Day 7 — SELinux** next.
