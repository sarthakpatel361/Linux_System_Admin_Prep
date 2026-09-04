# Day 2 — Networking (Full Study Edition, with Answers)

**Estimated time: 4–5 hours**

Day 1 was storage; today is everything between the NIC and the application actually being reachable. Product-company interviewers rarely ask "what does `nmcli` do" — they ask "a service is unreachable, walk me through how you find out why," and expect you to move through the network stack in a disciplined order instead of guessing. That escalation discipline is the real subject of today, with `nmcli`/bonding/routing as the tools you use to execute it. Each topic below opens with a **Quick Review**, a diagram where it clarifies the concept, a hands-on **Implementation**, and **Interview Questions with detailed answers** — attempt each question yourself before reading the answer.

---

## Topic 1: NetworkManager — nmcli/nmtui and Connection Profiles

### Quick Review
- RHEL 8/9 network config lives in **NetworkManager connection profiles**, not raw `network-scripts` interface files.
- A profile is a **named config object** bound to a device — you can have multiple profiles per NIC and switch between them.
- `nmcli` is the CLI; `nmtui` is the menu-driven equivalent for console-only access.
- Profile changes aren't live until you `nmcli con up` — editing a profile doesn't touch the running kernel state by itself.

### Quick Learning

RHEL 8/9 manage networking through NetworkManager, which stores configuration as connection profiles — this is a deliberate architectural split between "what should this interface's config be" (the profile, sitting in `/etc/NetworkManager/system-connections/`) and "what is the kernel actually doing right now" (live state, visible via `ip addr`/`ip route`). Understanding that split is what lets you reason correctly about a change that "didn't take effect" — often the profile was updated correctly but never activated, or an old profile is still active alongside the new one.

**Profile vs. live kernel state — where a change actually lives:**
```
  nmcli con mod "System eth0" ...            nmcli con up "System eth0"
  ────────────────────────────               ─────────────────────────
  Writes to the PROFILE                       Pushes profile config DOWN
  (/etc/NetworkManager/                       into the kernel — THIS is
   system-connections/*.nmconnection)         what makes `ip addr show`
                                               actually reflect the change

  ip addr show  right here  ──▶  still shows OLD config, because the
                                  profile edit alone never touched
                                  the kernel's live interface state
```

### Implementation (Learn by Applying)

**Scenario:** You're handed a fresh RHEL 9 VM with a single NIC on DHCP. The application team needs it converted to a static IP with a secondary management address, without ever losing SSH connectivity mid-change — a real production constraint.

```bash
nmcli connection show
nmcli device status
nmcli connection show "System eth0" | grep -E "ipv4|ipv6"

nmcli con mod "System eth0" ipv4.addresses 192.168.1.50/24
nmcli con mod "System eth0" ipv4.gateway 192.168.1.1
nmcli con mod "System eth0" ipv4.dns "8.8.8.8 1.1.1.1"
nmcli con mod "System eth0" ipv4.method manual
nmcli con mod "System eth0" +ipv4.addresses 192.168.1.51/24

nmcli con up "System eth0"

ip addr show
ip route show
ping -c2 192.168.1.1
```

**The zero-lockout trick worth internalizing:** before a risky network change over SSH, open a second SSH session first as a fallback path, or schedule a rollback: `echo "nmcli con up backup-dhcp" | at now + 5 minutes` — if the change breaks connectivity, the scheduled rollback fires and restores it even if you're locked out.

```bash
nmcli con reload
nmcli -t -f NAME,DEVICE,STATE con show
```

### Interview Questions — with Answers

**1. What's actually stored in a NetworkManager connection profile versus what's queried live from the kernel — if I run `ip addr show` right after `nmcli con mod` but before `nmcli con up`, what would I see?**

The profile stores the *intended* configuration — IP addresses, gateway, DNS, method, and metadata — as a file on disk, independent of whatever the interface is currently doing. `ip addr show` queries the kernel's live network state directly, which is a completely separate source of truth. So right after `nmcli con mod` but before `nmcli con up`, `ip addr show` would still show the interface's *old* configuration — the modification only updated the saved profile, and nothing has been pushed down to the kernel yet.

**2. You need to change a production server's IP over SSH with zero tolerance for an extended lockout. Walk me through your exact process, including your fallback plan if the change breaks connectivity.**

First, I'd open a second, independent SSH session to the server before touching anything, so I have a working session even if the change breaks the primary one. I'd stage the entire new configuration on the profile first (`nmcli con mod` for every setting needed) without activating it, then review it fully before applying. As a safety net, I'd schedule an automatic rollback (`at` job or `systemd-run --on-active`) that reverts to a known-good profile a few minutes out, so even a total lockout self-heals. Only then would I run `nmcli con up`, immediately verify from the second session, and cancel the scheduled rollback once confirmed working.

**3. What's the practical difference between `nmcli con reload` and `systemctl restart NetworkManager` — when would each actually be necessary?**

`nmcli con reload` tells NetworkManager to re-read connection profile files from disk without restarting the service itself or touching any currently active connections — it's used when you've manually edited a `.nmconnection` file outside of `nmcli` and need NetworkManager to become aware of that change. `systemctl restart NetworkManager` restarts the entire service, which can briefly disrupt all managed interfaces and is a much heavier-handed operation — I'd reserve that for genuine NetworkManager service issues (it's misbehaving, hung, or needs to pick up a package update), not for routine profile changes, where `con reload` is safer and sufficient.

**4. A server has two connection profiles for the same physical NIC — one DHCP, one static — and after a reboot it's coming up on the wrong one. What would you check?**

I'd check each profile's `connection.autoconnect` setting and `connection.autoconnect-priority` — if both profiles have autoconnect enabled, NetworkManager picks based on priority (and profile ordering/timing as a tiebreaker), so the "wrong" one activating usually means its priority is higher or the "right" one's autoconnect is disabled entirely. I'd run `nmcli connection show` to confirm which profile is actually marked active post-boot, then adjust `autoconnect-priority` on the intended profile to be higher, and consider explicitly disabling autoconnect on the unwanted profile to remove ambiguity entirely.

**5. Explain what `nmtui` gives you that `nmcli` doesn't, and describe a real scenario where you'd be forced to use it.**

`nmtui` doesn't offer any capability `nmcli` lacks — it's a menu-driven front-end over the same underlying NetworkManager functionality — but it removes the need to remember exact command syntax, which matters when you're under pressure or working from an environment without easy access to documentation. A real scenario: you're on an IPMI/iDRAC/console session after a botched network change locked out SSH entirely, working from a minimal, possibly slow remote console — `nmtui`'s guided menus reduce the chance of a typo compounding an already-stressful recovery, compared to constructing a multi-flag `nmcli` command from memory under pressure.

---

## Topic 2: Bonding/Teaming for Redundancy and Throughput

### Quick Review
- Bonding combines multiple physical NICs into **one logical interface**.
- `active-backup` (mode 1) — redundancy only, no switch config required, safe default.
- `802.3ad`/LACP (mode 4) — real throughput aggregation, **requires** matching switch-side LACP config.
- Verify with `/proc/net/bonding/bondX`, not just "interface is up."

### Quick Learning

The mode you choose depends entirely on the goal (redundancy vs throughput) and what the switch side actually supports — a mismatch between the server's bond mode and the switch's port configuration is the single most common "bond is up but useless" root cause. Know `active-backup` and `802.3ad` cold; the other modes are edge cases you can name but rarely deploy.

**Two NICs, two very different outcomes depending on switch agreement:**
```
  active-backup (mode 1)                  802.3ad / LACP (mode 4)
  ───────────────────────                 ───────────────────────
  eth0 ──▶ ACTIVE                         eth0 ──┐
  eth1 ──▶ standby (idle)                 eth1 ──┴──▶ AGGREGATED
                                                       (both carry
  No switch config needed.                            traffic together)
  eth0 fails ⇒ eth1 takes over
  automatically, no coordination           Switch ports MUST be
  required from network team.              configured for LACP too —
                                            if not, traffic on the
                                            "aggregated" bond is
                                            actually unreliable/dropped
                                            despite links showing "up"
```

### Implementation (Learn by Applying)

**Scenario:** Build an LACP bond for a database server where the network team has confirmed LACP is configured on both switch ports, then intentionally fail one leg to confirm failover.

```bash
nmcli con add type bond ifname bond0 con-name bond0 mode 802.3ad
nmcli con add type ethernet ifname eth0 master bond0 con-name bond0-eth0
nmcli con add type ethernet ifname eth1 master bond0 con-name bond0-eth1
nmcli con mod bond0 ipv4.addresses 192.168.1.60/24 ipv4.gateway 192.168.1.1 ipv4.method manual

nmcli con up bond0-eth0
nmcli con up bond0-eth1
nmcli con up bond0

cat /proc/net/bonding/bond0
```
Confirm: `Bonding Mode: IEEE 802.3ad Dynamic link aggregation`, both slaves `MII Status: up`, and — critically for LACP — `Actor Churn State`/`Partner Churn State` read `none`; churning means the switch isn't actually negotiating LACP correctly despite the physical link being up.

```bash
nmcli con down bond0-eth0
cat /proc/net/bonding/bond0 | grep -A2 "Currently Active"
ping -c4 192.168.1.1
nmcli con up bond0-eth0
```

### Interview Questions — with Answers

**1. You inherit a bond configured in `802.3ad` mode, but throughput testing shows no improvement over a single NIC, and the switch team says they never configured LACP on those ports. What's actually happening, and how do you confirm it from the server side alone?**

The server believes it's running LACP and is attempting to negotiate aggregation, but without matching switch-side configuration, the switch is treating each port as an independent, non-aggregated link — so traffic isn't actually being load-balanced across both NICs the way `802.3ad` mode intends, explaining the lack of throughput improvement. From the server side, `cat /proc/net/bonding/bond0` would show this clearly: look at `Actor Churn State` and `Partner Churn State` — if the partner (switch) side is churning or the bond never reports a stable `Partner Mac Address` matching expectations, that's confirmation the switch isn't participating in LACP negotiation at all, even though the physical `MII Status` shows both links up.

**2. Why is `active-backup` often the safer default choice even though it "wastes" a NIC's bandwidth compared to LACP?**

`active-backup` requires zero coordination with the network team — it works correctly the moment you configure it on the server, regardless of what the switch ports are doing, because failover is entirely a server-side decision. LACP's throughput benefit is real, but it depends entirely on correct, matching switch-side configuration that the SysAdmin often doesn't control and can't fully verify without cooperation from network engineering — a silent mismatch (like the scenario in question 1) means you believe you have aggregation and redundancy when you actually just have fragile, unaggregated links. For a server where redundancy matters more than the throughput gain, or where you can't be 100% certain the switch side is correctly configured, `active-backup`'s simplicity and lack of cross-team dependency makes it the safer default.

**3. Walk me through exactly how you'd test that a bond genuinely fails over correctly, before you trust it in production — not just "it's configured," but proof it works.**

I'd start a continuous connectivity test (sustained `ping` with tight intervals, or better, an active TCP session/file transfer) that I can watch in real time, then deliberately bring down one bond member (`nmcli con down bond0-eth0`, or physically unplug it in a lab) while that traffic is running, and confirm the ping/transfer continues with at most a brief, sub-second interruption rather than a hard failure. I'd check `/proc/net/bonding/bond0` before and after to confirm the "Currently Active Slave" actually changed (for active-backup) or that the remaining link is still carrying traffic (for LACP), not just that the bond interface itself still shows "up" — an interface can report up while silently not passing traffic if the failover logic isn't actually working. Finally, I'd restore the failed link and confirm the bond returns to its normal, fully-redundant state rather than staying degraded.

**4. What's the difference between bonding and teaming (the older `teamd`-based approach) on RHEL, and which does Red Hat currently recommend?**

Both accomplish the same fundamental goal — combining multiple NICs into one logical interface — but bonding is the older, kernel-driver-based implementation (`bonding` kernel module), while teaming (`teamd`) was introduced as a more modern, userspace-daemon-based alternative with a more flexible/pluggable architecture for load-balancing and link-monitoring logic. In recent RHEL releases (8 and 9), Red Hat has actually shifted back toward recommending **bonding** as the primary supported method, with teaming considered mature but not the forward direction of active development — worth confirming against current RHEL documentation since Red Hat's recommendation has shifted over releases, but as of RHEL 8/9, bonding via NetworkManager is the standard path.

**5. If `/proc/net/bonding/bond0` shows both slaves as `MII Status: up` but the bond as a whole isn't passing traffic, what layers would you check next?**

`MII Status: up` only confirms the physical link layer is electrically/optically up — it says nothing about IP configuration, routing, or higher-layer connectivity. I'd check the bond interface's IP configuration itself (`ip addr show bond0`) to confirm it actually has the expected address, then `ip route show` to confirm routing is correctly pointed through the bond, then basic reachability tests (`ping` the gateway) to isolate whether the problem is at the bond/link layer at all versus a completely separate issue (firewall, wrong VLAN, switch-side port configuration not matching the bond mode as in question 1). I'd also check `ethtool bond0` and the individual slave interfaces for speed/duplex mismatches, and confirm the bond's assigned VLAN (if any) actually matches what the switch ports are trunking/access-configured for.

---

## Topic 3: Connectivity Troubleshooting — The Escalation Order

### Quick Review
- "It's not working" is not a diagnosis — move **layer by layer, outward** from the host.
- Order: IP → route → gateway reachability → external reachability → DNS → port reachability → local listener check → packet capture.
- Skipping steps wastes time investigating the wrong layer.

### Quick Learning

The disciplined approach moves strictly layer by layer, outward from the host — this order matters because each step rules out an entire category of causes before you spend time on the next one. Skipping steps is how people spend 40 minutes staring at firewall rules when the actual problem was a missing default route the whole time.

**The escalation order as a flowchart:**
```
  ┌─────────────────────┐
  │ 1. ip addr show        │  No IP? ──▶ DHCP/static config problem. STOP HERE.
  └──────────┬───────────┘
             ▼ (have IP)
  ┌─────────────────────┐
  │ 2. ip route show       │  No route? ──▶ missing/wrong default route. STOP HERE.
  └──────────┬───────────┘
             ▼ (have route)
  ┌─────────────────────┐
  │ 3. ping gateway         │  Fails? ──▶ local L2/L3 problem (switch port, VLAN,
  └──────────┬───────────┘             cable, ARP). STOP HERE.
             ▼ (gateway OK)
  ┌─────────────────────┐
  │ 4. ping external IP    │  Fails? ──▶ upstream routing/firewall issue between
  └──────────┬───────────┘             here and there. STOP HERE.
             ▼ (external IP OK)
  ┌─────────────────────┐
  │ 5. dig / resolv.conf   │  Fails? ──▶ DNS problem specifically — NOT connectivity.
  └──────────┬───────────┘             STOP HERE.
             ▼ (DNS OK)
  ┌─────────────────────┐
  │ 6. curl/nc to PORT     │  Hangs/refused? ──▶ firewall or app not listening
  └──────────┬───────────┘             remotely. Check step 7 if it's YOUR box.
             ▼ (port reachable)
  ┌─────────────────────┐
  │ 7. ss -tulnp (local)   │  Confirms something IS actually bound/listening
  └──────────┬───────────┘
             ▼ (all above pass, still broken)
  ┌─────────────────────┐
  │ 8. tcpdump               │  Last resort — packet-level truth when every
  └─────────────────────┘  higher-level tool says "should be fine"
```

### Implementation (Learn by Applying)

**Scenario:** An application team reports "we can't reach our API server on port 8443." Break one thing deliberately, then diagnose it blind, following the order above.

```bash
# Pick ONE to break:
ip route del default
# or: mv /etc/resolv.conf /etc/resolv.conf.bak
# or: firewall-cmd --add-rich-rule='rule family="ipv4" port port="8443" protocol="tcp" reject' --timeout=600
```

```bash
ip addr show
ip route show
ip route get 8.8.8.8

ping -c2 $(ip route show default | awk '{print $3}')
ping -c2 8.8.8.8

dig api.example.com
nslookup api.example.com
cat /etc/resolv.conf

curl -v telnet://api.example.com:8443
nc -zv api.example.com 8443

ss -tulnp | grep 8443

tcpdump -i any port 8443 -nn
```

Fix and confirm:
```bash
ip route add default via <gateway_ip>
mv /etc/resolv.conf.bak /etc/resolv.conf
firewall-cmd --reload
```

### Interview Questions — with Answers

**1. Walk me through your exact troubleshooting order for "we can't reach the API server" — not just naming commands, the actual sequence and why that sequence.**

I start closest to the host and work outward, because each layer rules out a whole category of causes before I invest time in the next: confirm the host has an IP (`ip addr`), confirm it has a route (`ip route`), confirm it can reach the immediate gateway (rules out local L2/switch-port issues), confirm it can reach something external by IP (rules DNS in or out as a variable), confirm DNS actually resolves the hostname in question, confirm the specific port is reachable at the application layer, and only then — if everything above checks out and it's still broken — go to packet capture. The reasoning behind the order specifically: reachability by IP before DNS isolates whether it's a name-resolution problem versus a genuine routing/connectivity problem, and testing the gateway before testing anything external isolates local-network issues from upstream ones.

**2. `ping` to an external IP works, but `ping` to a hostname fails. What does that immediately tell you, and what's your next command?**

Since raw IP connectivity works, routing, the gateway, and the network path itself are all fine — the failure is isolated specifically to name resolution. My next command would be `dig <hostname>` (or `nslookup`) to see exactly how resolution is failing — whether it's timing out entirely (suggesting the DNS server itself is unreachable, which I'd cross-check against `cat /etc/resolv.conf` to confirm the configured DNS server's IP), or returning an explicit error (NXDOMAIN, servfail) which points to a DNS server-side or zone-configuration issue rather than a connectivity problem on my end at all.

**3. `curl -v` to a specific port hangs indefinitely rather than immediately refusing the connection. What does hanging (versus an instant "connection refused") tell you about where the problem likely is?**

An instant "connection refused" means a TCP RST came back immediately — something at that IP actively responded, meaning the host is reachable but nothing is listening on that port (or a firewall explicitly rejected it with a reset). Hanging indefinitely, with no response at all, almost always means packets are being silently dropped somewhere along the path — most commonly a firewall configured to DROP rather than REJECT, either on the destination host, an intermediate network firewall, or a security group/ACL — the client just waits for a TCP handshake response that's never coming and never being explicitly refused either.

**4. A service's port shows as listening in `ss -tulnp` on the server itself, but a remote client still can't connect. List every layer between those two facts that could be the actual cause.**

Starting from the server outward: the service might only be bound to `127.0.0.1`/localhost rather than `0.0.0.0` or the server's actual external interface (visible in `ss` output's local address column), meaning it's genuinely listening but only reachable from the server itself. Next, the host's own firewall (firewalld/iptables/nftables) could be blocking the port for external sources even though the process is happily listening locally. Beyond the host, a network-level firewall, security group (in cloud environments), or ACL sitting between the client and server could be blocking the traffic before it even arrives. Finally, routing issues on either the client or server side, or on intermediate hops, could prevent the packets from ever reaching the server's interface at all — "listening locally" only proves the application layer is fine, it says nothing about anything below or beyond it.

**5. When do you reach for `tcpdump` instead of the higher-level tools, and what would you actually be looking for in the capture that the higher-level tools couldn't already tell you?**

I reach for `tcpdump` when every higher-level check (routing, DNS, port reachability tools, local listener confirmation) has passed and the problem is still unresolved, or when the symptom is ambiguous/intermittent in a way that suggests something subtler than a hard configuration issue — asymmetric routing, unexpected retransmissions, or packets arriving but with unexpected flags/content. What `tcpdump` reveals that higher-level tools can't: whether packets are actually leaving the interface at all (versus being silently dropped before even hitting the wire), whether responses are coming back but being dropped/ignored somewhere in the return path (asymmetric routing is a classic cause), the exact TCP flags exchanged (a SYN sent with no SYN-ACK ever returned tells a very different story than a SYN-ACK arriving but the connection still failing afterward), and timing/sequence details that explain intermittent or degraded-but-not-fully-broken behavior that a simple pass/fail tool like `curl` can't fully characterize.

---

## Topic 4: Deeper Diagnostics — ss, Routing Internals, and MTU/Duplex Issues

### Quick Review
- `ss` reveals connection-state health beyond simple up/down — e.g. TIME-WAIT churn, stuck SYN-SENT.
- MTU mismatches cause a specific symptom: small packets succeed, large transfers hang or fail.
- `ping -M do -s <size>` finds the actual working MTU by disabling fragmentation.
- A firewall that silently DROPs (vs REJECTs) ICMP is a common cause of broken Path MTU Discovery.

### Quick Learning

Beyond basic reachability, product-company interviews probe whether you can diagnose *degraded* (not fully down) network conditions. MTU mismatches between hops cause a specific, recognizable symptom: small packets succeed while large transfers hang or fail, because fragmentation-needed ICMP messages are often blocked somewhere in the path — a symptom worth recognizing on sight rather than re-deriving from scratch during an interview or an incident.

**Why small packets work but large transfers stall:**
```
  Client ──(small ping, 64 bytes)──▶ [Path MTU: 1500] ──▶ Server   ✔ arrives fine

  Client ──(large packet, needs fragmentation)──▶ [Hop with MTU 1400]
              │
              ▼
        Router SHOULD send back:
        "ICMP: Fragmentation Needed, use MTU 1400"
              │
              ▼
        ...but a firewall along the path is configured to
        DROP ICMP entirely (common "security hardening" mistake)
              │
              ▼
        Client never learns the real path MTU ⇒ keeps sending
        oversized packets ⇒ they silently vanish ⇒ transfer hangs,
        while ping (small packets) keeps working perfectly fine
```

### Implementation (Learn by Applying)

**Scenario:** A file transfer to a remote host consistently stalls partway through, even though basic connectivity works fine.

```bash
ping -c4 remotehost

ping -M do -s 1472 -c4 remotehost     # 1472 + 28 header = 1500, the standard MTU
ping -M do -s 1400 -c4 remotehost
ping -M do -s 1300 -c4 remotehost

ip link show eth0 | grep mtu

tracepath remotehost                   # Reports actual path MTU discovered per hop
```

```bash
ss -s
ss -tan state time-wait | wc -l
ss -tan state syn-sent
```

### Interview Questions — with Answers

**1. A user says small pings work fine but large file transfers to the same host hang. What's your hypothesis, and how do you confirm it without guessing?**

My working hypothesis is an MTU mismatch somewhere along the path combined with ICMP being blocked, preventing Path MTU Discovery from working correctly — small pings fit within the smallest MTU on the path, but larger packets required by a file transfer need fragmentation-needed feedback that's never arriving. I'd confirm it methodically rather than guessing: `ping -M do -s <size>` with decreasing sizes to find exactly where packets start failing (the size just above the working threshold tells you the actual constrained MTU), and `tracepath` to see the discovered path MTU per hop directly, which either confirms the hypothesis or rules it out in favor of investigating something else.

**2. What's the practical difference in symptoms between a firewall that REJECTs a connection versus one that DROPs it silently — how would you tell which one you're dealing with from the client side?**

A REJECT sends back an explicit response (a TCP RST for TCP, or an ICMP "port unreachable" for UDP), so the client gets immediate, clear feedback that the connection was refused — tools like `curl` or `nc` return quickly with an explicit "connection refused" error. A DROP silently discards the packet with no response at all, so the client just waits, and the connection attempt eventually times out on its own client-side timeout — from the client, this shows up as a long hang rather than an instant failure. Practically: time how long it takes to fail — instant failure with an explicit error points to REJECT (or the destination actively refusing), while a slow timeout with no immediate feedback points to a silent DROP somewhere in the path.

**3. You see a very high and growing count of TCP connections in `TIME-WAIT` state on a busy web server. Is this necessarily a problem, and what would you check to decide?**

Not necessarily — `TIME-WAIT` is a normal, expected part of TCP's connection teardown (ensuring delayed/duplicate packets from a closed connection don't get misattributed to a new one), and a busy server handling many short-lived connections will naturally accumulate a meaningful count. Whether it's actually a problem depends on trend and scale relative to the system's tuned limits: I'd check if the count is stable/proportional to traffic versus climbing unboundedly, check `sysctl net.ipv4.tcp_tw_reuse`/`tw_recycle`-equivalent tuning and ephemeral port range exhaustion (a genuinely excessive `TIME-WAIT` count can exhaust available source ports for new outbound connections), and correlate with whether the server is actually experiencing connection failures or just carrying a large but harmless backlog of recently-closed sockets.

**4. Explain what Path MTU Discovery is supposed to do automatically, and describe a real-world scenario (hint: think VPNs/tunnels) where it commonly breaks.**

Path MTU Discovery is meant to let a sender automatically learn the smallest MTU along the entire path to a destination by sending packets with the "don't fragment" flag set and relying on routers along the way to respond with an ICMP "fragmentation needed" message whenever a packet is too large for the next hop — the sender then adjusts its packet size downward accordingly, all without manual intervention. It commonly breaks with VPNs/tunnels (IPsec, GRE, overlay networks) because tunnel encapsulation adds extra header overhead, effectively reducing the usable MTU inside the tunnel below the physical network's standard 1500 — if a firewall along the path blocks the ICMP messages needed for discovery to work (a common hardening misconfiguration), traffic through the tunnel silently stalls on anything larger than the reduced effective MTU, producing exactly the "small packets fine, large transfers hang" symptom from the earlier lab.

**5. Two servers that used to communicate fine suddenly show intermittent, unpredictable slowness — not a hard failure. What's your triage order between "check the network path" and "check the application/host itself," and how do you decide where to start?**

I'd start with quick, cheap checks on both sides in parallel rather than committing fully to one theory first: on the host/application side, a fast look at `top`/`vmstat`/`iostat` on both servers rules out (or confirms) resource contention as the cause in under a minute; on the network side, `mtr` between the two hosts run over a period long enough to catch the intermittent behavior shows whether packet loss or latency spikes correlate with the slowness. The decision about where to dig deeper follows the evidence: if `mtr` shows clean, consistent low-loss results throughout the observed slowness, the problem is very likely host/application-side and I'd focus there next (application logs, GC pauses, disk I/O); if `mtr` shows loss or latency spikes correlating with the reported slow periods, I'd focus on the network path — potentially involving the network team if the problem is beyond a hop I control.

---

## Topic 5 (Bonus — Advanced): Routing Tables & Policy Routing on Multi-Homed Hosts

### Quick Review
- A host with more than one NIC/gateway needs routing decisions beyond a single default route.
- The kernel consults routing **tables** in an order governed by **rules** (`ip rule`), not just the main table.
- Policy routing lets traffic be routed differently based on source address, not just destination — essential for multi-homed servers.
- `ip route get <dest>` shows exactly which route will actually be used, without guessing.

### Quick Learning

A server with a single NIC only ever needs one routing decision: "send everything through the default gateway." A multi-homed server — say, one NIC for application traffic, another for backup/management traffic, each on different subnets with different gateways — needs the kernel to make a more nuanced decision, and the default routing table alone often can't express "traffic FROM this source IP should go out THIS interface," which is exactly the gap policy routing (`ip rule` + multiple routing tables) fills. This is a genuinely advanced topic that separates someone who's only run `ip route add default` on a single-NIC box from someone who's actually operated multi-homed production servers.

**Why a single default route isn't enough on a multi-homed host:**
```
        RHEL Server
     ┌───────┴───────┐
   eth0                eth1
  10.0.1.5/24         10.0.2.5/24
  gw 10.0.1.1         gw 10.0.2.1
  (application net)   (backup net)

  With only ONE default route (say, via eth0's gateway):
     - Traffic destined for the backup network works fine (eth1 subnet, direct)
     - Traffic SOURCED from eth1 but destined elsewhere tries to route
       out via eth0's default gateway anyway ⇒ asymmetric routing ⇒
       return traffic often gets dropped by the OTHER side's firewall,
       which doesn't expect a reply from an unexpected source interface

  Policy routing fixes this: "traffic sourced FROM 10.0.2.5 uses
  the eth1 routing TABLE (its own default gateway), regardless of
  what the MAIN table's default route says"
```

### Implementation (Learn by Applying)

**Scenario:** A server has two NICs on separate subnets with separate gateways. Traffic originating from the second NIC needs to route back out through its own gateway, not the primary default route — configure policy routing to fix asymmetric routing.

```bash
# Baseline: see the current single routing table and its one default route
ip route show

# Add a route to the second subnet's gateway in a SEPARATE table
echo "200 backupnet" >> /etc/iproute2/rt_tables
ip route add default via 10.0.2.1 dev eth1 table backupnet
ip route add 10.0.2.0/24 dev eth1 src 10.0.2.5 table backupnet

# Add a RULE telling the kernel: traffic sourced from 10.0.2.5 uses the backupnet table
ip rule add from 10.0.2.5 table backupnet
ip rule show

# Verify the kernel now makes the correct decision for traffic FROM that source
ip route get 8.8.8.8 from 10.0.2.5

# Persist via NetworkManager (per-connection routing rules, not just ip commands that vanish on reboot)
nmcli con mod bond0-eth1 +ipv4.routing-rules "priority 100 from 10.0.2.5 table 200"
nmcli con mod bond0-eth1 ipv4.route-table 200
```

### Interview Questions — with Answers

**1. Why does a multi-homed server sometimes need more than just a single default route to function correctly?**

A single default route only tells the kernel where to send traffic when the destination doesn't match any more specific route — it says nothing about which *outbound path* to prefer based on where the traffic originated. On a multi-homed server, traffic that's sourced from a secondary interface's IP still gets routed according to the single default route's gateway unless told otherwise, which can send replies back out the wrong physical interface — breaking anything that expects symmetric routing (many stateful firewalls, for instance, drop return traffic arriving via an unexpected path). Multiple routing tables plus policy rules (`ip rule`) let you express "traffic from THIS source uses THIS table/gateway," which a single default route fundamentally can't.

**2. What's the actual command to see which route the kernel will pick for a given destination and source, without guessing or trial-and-error?**

`ip route get <destination> from <source>` — this asks the kernel to actually perform its routing lookup for that exact destination/source combination and report back which route, table, and interface it would use, rather than me reading multiple `ip route show table X` outputs and mentally simulating the lookup order myself. It's the single fastest way to confirm policy routing is (or isn't) working as intended.

**3. Explain asymmetric routing and why it's a common cause of "intermittent" connectivity problems that are actually 100% consistent once you understand the cause.**

Asymmetric routing happens when outbound traffic to a destination takes one path, but the return traffic from that destination takes a *different* path back — for example, a request going out via eth0's gateway but a reply arriving in via eth1 because of how the remote side or an intermediate router decided to route it. Many stateful firewalls (including the local host's own connection tracking, or an intermediate network firewall) expect to see both directions of a TCP conversation on the same path/interface, and drop return traffic that "doesn't belong" to a tracked connection when it arrives somewhere unexpected — this looks intermittent from a user's perspective because it might only affect specific connections or specific traffic patterns, but it's actually a deterministic outcome of the routing configuration, not genuinely random.

**4. What's the difference between `ip route` and `ip rule` — why do you need both to implement policy routing, and what does each one actually control?**

`ip route` populates the entries *within* a specific routing table — the actual destination-to-gateway/interface mappings. `ip rule` controls which routing table the kernel even *consults* for a given packet, based on criteria like source address, in what priority order — it's a layer above routing tables that decides "for this kind of traffic, look in THIS table" before the destination lookup within that table even happens. You need both because `ip route` alone only lets you populate one table's worth of destination-based decisions, while `ip rule` is what actually enables source-based (or other criteria-based) routing decisions by directing different traffic to different tables in the first place.

**5. If you configure policy routing with raw `ip rule`/`ip route` commands and it works perfectly, but the configuration disappears after a reboot, what did you forget, and how do you fix it properly on RHEL?**

Raw `ip rule add` and `ip route add` commands only modify the kernel's live routing state — they aren't persistent configuration, so they vanish on reboot exactly like any other unsaved runtime kernel state. The correct, persistent approach on RHEL is configuring the same policy routing through NetworkManager connection profiles, using properties like `ipv4.routing-rules` and `ipv4.route-table` (as shown in the lab), which NetworkManager reapplies automatically every time the connection is activated — including at every boot — rather than relying on manually re-running `ip` commands or maintaining separate legacy `/etc/sysconfig/network-scripts/rule-*`/`route-*` files, which is the older RHEL 7-style approach still supported but not the current recommended pattern.

---

**End of Day 2.** You should now be able to change network configuration over SSH without risking a lockout, stand up and genuinely failover-test a bond, work a connectivity outage through the correct layer-by-layer escalation instead of guessing, recognize the specific symptom signature of MTU and connection-state problems, and — the advanced capstone — reason correctly about policy routing on a multi-homed host, including why asymmetric routing produces what looks like intermittent failures but is actually fully deterministic.

Proceed to **Day 3 — Systemd & Service Management** next.
