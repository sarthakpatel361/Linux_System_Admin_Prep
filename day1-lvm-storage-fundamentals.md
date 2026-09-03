# Day 1 — LVM & Storage Fundamentals

**Estimated time: 2–3 hours** (this assumes you actually run every lab on a real or virtual RHEL 9 box, not just read the commands)

This day covers the full LVM lifecycle the way a product-company SysAdmin actually touches it in production: build it, grow it live, understand where shrinking breaks down, protect it with snapshots, and reason about what makes it *resilient* versus just "working today." Given your SAN/Power Systems background, pay close attention to the resilience and thin-provisioning sections — this is where your existing mental model (LUNs, mirroring, thin pools on the array side) maps almost directly onto LVM concepts, and interviewers at product companies love probing that mapping.

---

## Topic 1: LVM Architecture — PV, VG, LV

### Quick Learning

LVM adds an abstraction layer between raw disks and filesystems so storage can be resized, moved, and pooled without caring about the underlying physical layout. The chain is: **Physical Volume (PV)** — a raw disk or partition initialized for LVM use → **Volume Group (VG)** — a pool combining one or more PVs into a single allocatable space → **Logical Volume (LV)** — a slice of that pool you actually format and mount, functionally equivalent to a "virtual disk." Internally, a VG's space is broken into fixed-size **Physical Extents (PEs)**; an LV is just a set of PEs mapped in sequence. This PE-mapping is *why* LVM can grow, shrink, and move volumes on the fly — you're just reassigning extent pointers, not physically relocating data (except during `pvmove`).

### Implementation (Learn by Applying)

**Scenario:** You've been handed a fresh RHEL 9 VM with three unused 10GB disks (`/dev/sdb`, `/dev/sdc`, `/dev/sdd`). Build an LVM stack that pools all three into one VG, and carve out a 15GB LV for an application that will need to grow later.

Work through it:

```bash
# Confirm the disks are unused and unpartitioned
lsblk -f

# Initialize each disk as a PV
pvcreate /dev/sdb /dev/sdc /dev/sdd
pvs

# Pool them into a single VG
vgcreate vg_app /dev/sdb /dev/sdc /dev/sdd
vgs -o +vg_free_count,vg_extent_count

# Carve out a 15G LV, leaving headroom in the VG for future growth
lvcreate -n lv_app -L 15G vg_app
lvs -o +devices          # Notice it may span more than one PV depending on extent allocation

# Format and mount it
mkfs.xfs /dev/vg_app/lv_app
mkdir -p /appdata
mount /dev/vg_app/lv_app /appdata
blkid /dev/vg_app/lv_app
echo "UUID=$(blkid -s UUID -o value /dev/vg_app/lv_app) /appdata xfs defaults,nofail 0 0" >> /etc/fstab
mount -a   # Validate fstab syntax immediately — don't wait for the next reboot to find out it's broken
```

Deliberately leave ~15GB unallocated in the VG rather than using `100%FREE` — that headroom is what Topic 2 will use.

### Interview Questions

1. Walk me through what actually happens at each layer — PV, VG, LV — when I run `df -h` and see 15GB available on `/appdata`. Where does that number actually come from?
2. Why would a team choose LVM over just formatting a disk directly, especially in a cloud/VM environment where the disk itself can often already be resized at the hypervisor level?
3. If I run `lvcreate -l 100%FREE`, what specifically am I giving up compared to leaving unallocated space in the VG? *(hint: think about what operations become harder later)*
4. You inherit a server where `vgs` shows a VG with 0% free space, but the LV mounted from it shows only 40% used in `df -h`. How do you explain that gap, and what would you check first?
5. What's the actual difference between a Physical Extent size mismatch across PVs in the same VG — can you even mix PVs with different PE sizes in one VG?

---

## Topic 2: Extending Storage Without Downtime

### Quick Learning

Growing storage live is one of LVM's core selling points over raw partitions, and it's a two-layer operation people often botch by doing only half of it: you must extend the **LV** (claim more extents from the VG) *and* grow the **filesystem** sitting on top of it (tell XFS/ext4 to actually use the new space) — extending the LV alone leaves the filesystem still reporting the old size. `lvextend -r` does both in one atomic-feeling step, but you should understand the two separate operations underneath it, because interviewers will ask you to explain them independently.

### Implementation (Learn by Applying)

**Scenario:** The application team on `/appdata` (from Topic 1) is filling up faster than projected and needs 10 more GB *today*, with zero downtime. You have exactly the headroom you left unallocated in Topic 1.

```bash
# Confirm there's free space in the VG to draw from
vgs vg_app

# Extend the LV by 10G and grow the filesystem in the same command
lvextend -r -L +10G /dev/vg_app/lv_app

# Verify at every layer — don't just trust the command succeeded
lvs vg_app/lv_app
df -hT /appdata
xfs_info /appdata | head -5
```

Now simulate the scenario where the VG itself is out of headroom — a more realistic "it's 2 AM and we need space now" situation:

```bash
# Attach a brand new disk to the VM/host first (in a real scenario: SAN team presents a new LUN, or you add a virtual disk)
pvcreate /dev/sde
vgextend vg_app /dev/sde
vgs vg_app                     # Confirm the new free space landed in the VG
lvextend -r -L +5G /dev/vg_app/lv_app
```

Finally, understand *where* the new extents physically landed:

```bash
lvs -o +devices /dev/vg_app/lv_app
```
If the output shows the LV now spans across `/dev/sde` as well as the original disks, that's expected and fine for most workloads — but note it, because it becomes relevant when a disk fails (Topic 5).

### Interview Questions

1. A junior admin runs `lvextend -L +10G /dev/vg_app/lv_app` (no `-r`) and tells you "it didn't work, `df -h` still shows the old size." What actually happened, and what two commands do they need to run to fully fix it?
2. Can you grow an XFS filesystem while it's mounted and actively being written to? What about ext4? Are there any circumstances where you'd need to unmount first?
3. The VG has zero free extents and there's no new physical disk available to add. What are your actual options to get the application more space, in order of how disruptive they are?
4. Walk me through your debugging steps: a user says "LVM won't extend" — the `lvextend` command just hangs or errors out. What do you check, in what order?
5. If an LV spans multiple PVs after growth (like in the lab above), does that create any performance implications, positive or negative, versus an LV that lives entirely on one PV?

---

## Topic 3: Shrinking & Reducing — Where It Breaks Down

### Quick Learning

Shrinking is the operation most RHCSA-level admins have *never actually done* in production, because it's genuinely riskier: you must shrink the **filesystem first**, then the **LV** — the reverse order of extending — because shrinking the LV before the filesystem risks truncating data blocks the filesystem still thinks it owns. Critically: **XFS cannot shrink at all, by design** (a deliberate tradeoff Red Hat made for XFS's architecture). If a volume needs to shrink and it's XFS, your only real options are backup-recreate-restore onto a smaller volume, or live with the current size. ext4 *can* shrink, but only while unmounted.

### Implementation (Learn by Applying)

**Scenario:** You provisioned an ext4-formatted 20GB LV for a logging volume that turned out to be dramatically over-sized — you need to reclaim 10GB back to the VG for other use, and you're allowed a short maintenance window to unmount it.

```bash
# Build the scenario: a separate ext4 LV to practice shrinking on
lvcreate -n lv_logs -L 20G vg_app
mkfs.ext4 /dev/vg_app/lv_logs
mkdir -p /logs
mount /dev/vg_app/lv_logs /logs

# Shrinking requires unmounting first
umount /logs

# ALWAYS check filesystem consistency before resizing — never skip this
e2fsck -f /dev/vg_app/lv_logs

# Shrink the filesystem FIRST, to a size smaller than your target LV size (leave margin)
resize2fs /dev/vg_app/lv_logs 9G

# THEN shrink the LV to match
lvreduce -L 10G /dev/vg_app/lv_logs

# Remount and verify
mount /dev/vg_app/lv_logs /logs
df -hT /logs
```

Now prove to yourself why XFS is different — try (and fail) to shrink one:

```bash
lvcreate -n lv_xfs_test -L 10G vg_app
mkfs.xfs /dev/vg_app/lv_xfs_test
mount /dev/vg_app/lv_xfs_test /mnt
umount /mnt
xfs_growfs -n /mnt 2>&1 || true   # There is no xfs_shrinkfs — confirm this for yourself
```

### Interview Questions

1. Why can't XFS shrink, architecturally? If you don't know the exact internal reason, at minimum: what's your practical workaround when a business asks you to shrink an over-provisioned XFS volume?
2. What happens — actually, mechanically — if someone shrinks the LV *before* shrinking the filesystem? Why is the order non-negotiable for ext4?
3. When would you actually recommend shrinking storage in production versus just leaving over-provisioned space alone? Is shrinking always worth the risk?
4. A colleague wants to shrink a live, mounted, actively-written-to ext4 volume without a maintenance window, to avoid downtime. What do you tell them?
5. If you must "shrink" an XFS volume with zero shrink support, walk me through the actual migration approach you'd use, step by step, including how you'd minimize the downtime window.

---

## Topic 4: Snapshots & Copy-on-Write

### Quick Learning

An LVM snapshot doesn't clone your data — it creates a lightweight tracking volume that captures blocks *only as they change* on the origin (Copy-on-Write). The snapshot's declared size isn't "a copy of the volume," it's overflow room sized for expected *write volume* during the snapshot's life. If writes to the origin exceed the snapshot's allocated space before you remove it, LVM **invalidates** the snapshot outright — it becomes unusable, silently, unless you're actively monitoring fill percentage. This is the single most common "why did my pre-patch backup snapshot fail" root cause in real incidents.

### Implementation (Learn by Applying)

**Scenario:** You're about to run a risky config change on the `lv_app` volume from Topic 1/2. Take a snapshot as a rollback point, simulate writes filling the snapshot's overflow space, and practice both a successful rollback and recognizing an invalidated snapshot.

```bash
# Take a snapshot before the "risky change" — size it deliberately small to force the failure scenario later
lvcreate -s -n lv_app_snap -L 500M /dev/vg_app/lv_app
lvs -o +snap_percent vg_app

# Simulate the "risky change": write some data to the origin
dd if=/dev/zero of=/appdata/testfile bs=1M count=100
lvs -o +snap_percent vg_app     # Watch the snapshot fill percentage climb

# Scenario A: rollback while the snapshot is still healthy
umount /appdata
lvconvert --merge vg_app/lv_app_snap
mount /dev/vg_app/lv_app /appdata    # Confirm testfile is gone — the rollback worked
```

Now force an invalidation to see what it actually looks like:

```bash
lvcreate -s -n lv_app_snap2 -L 100M /dev/vg_app/lv_app
dd if=/dev/zero of=/appdata/bigfile bs=1M count=500   # Deliberately exceed the snapshot's overflow capacity
lvs -o +snap_percent vg_app          # Look for the snapshot showing invalid/100% and no longer usable
lvdisplay /dev/vg_app/lv_app_snap2   # Check the "Status" field — this is what "invalidated" looks like in practice
```

Clean up:
```bash
lvremove vg_app/lv_app_snap2
```

### Interview Questions

1. Explain Copy-on-Write in your own words — specifically, when a write happens to the origin volume, what exactly gets copied where, and when?
2. How do you size a snapshot correctly for a production maintenance window? What factors go into that number?
3. A snapshot shows 100% full in `lvs -o +snap_percent` output. Is the data on the origin volume at risk? Is the snapshot itself still usable for rollback?
4. What's the operational difference between using an LVM snapshot as a rollback mechanism versus using it as a backup source (e.g., mounting it read-only and copying data off before removing it)?
5. Walk me through your monitoring approach: how would you make sure nobody gets paged at 3 AM because a snapshot silently invalidated during an overnight batch job?

---

## Topic 5: Resilience — Surviving a Disk Failure

### Quick Learning

"Resilience" in LVM has two very different meanings people conflate: **LVM-level mirroring/RAID** (`lvcreate -m1` or `--type raid1`) actually protects against a single physical disk failure by keeping redundant copies — but it's rarely used in product companies because SAN/cloud block storage (which you already have direct experience with) typically provides redundancy at a layer below LVM already, making LVM-level mirroring redundant-on-redundant and a performance cost with no real benefit. What *is* commonly used and expected knowledge: designing VGs so a single disk loss doesn't take down every LV in the group, monitoring PV/disk health proactively (before the kernel starts throwing I/O errors), and knowing the actual recovery procedure when a PV in an active VG does fail.

### Implementation (Learn by Applying)

**Scenario:** You're asked to explain and demonstrate what happens when a PV backing an active VG fails, and how you'd recover — plus set up proactive monitoring so you're never caught by surprise.

Simulate a PV going missing (safe on a lab VM — do NOT attempt this pattern on any volume with real data):

```bash
# Check current PV/VG health as a baseline
pvs
vgs -o +vg_missing_pv_count 2>/dev/null || vgs

# Detach a disk from the VM at the hypervisor level (VirtualBox/VMware/KVM), or if truly just practicing safely, use:
pvs -a       # -a shows ALL devices LVM can see, including ones not currently part of an active VG — useful for spotting a PV that's dropped out
```

After detaching a disk that belongs to `vg_app` (via your hypervisor), reboot or rescan:

```bash
vgscan
pvscan
vgs vg_app                  # Note the "Attr" column showing a "p" flag for partial/missing PV, or VG marked partial
vgdisplay vg_app | grep -i "cur pv\|act pv"   # Compare current PVs vs active PVs — a gap here confirms the missing disk
```

**Recovery approach** (conceptual — practice reading this, since actual recovery syntax depends heavily on which LVs had extents on the failed disk):
```bash
# If you have a replacement disk ready:
pvcreate /dev/sdX_new
vgextend vg_app /dev/sdX_new
vgreduce --removemissing vg_app       # Remove the dead PV's references — data on unaffected LVs survives; LVs that had extents ONLY on the dead disk are lost unless mirrored
```

Now set up **proactive** monitoring so you catch a failing disk before it fully dies:
```bash
smartctl -a /dev/sdb | grep -E "Reallocated_Sector_Ct|Current_Pending_Sector|SMART overall-health"
# A cron job or monitoring agent checking these values and alerting on non-zero/climbing counts is the actual production pattern
```

### Interview Questions

1. In a SAN or cloud-block-storage environment, why might a team deliberately choose *not* to use LVM-level mirroring even though it's available? *(this one draws directly on your storage background — think about where redundancy is already happening)*
2. A VG shows one of its PVs as missing. Walk me through exactly how you determine which LVs are actually affected versus which ones are untouched.
3. What's the difference between `vgreduce --removemissing` with and without the `--force` flag, and why does that distinction matter for how much data you might lose?
4. How would you design a VG layout, from the start, to minimize the blast radius of a single disk failure — i.e., so losing one disk doesn't take out every LV in the group?
5. Describe your monitoring/alerting approach for storage health in general — not just LVM, the full stack from SMART data up through filesystem free space. What would wake you up at night, and what would just email you in the morning?

---

## Topic 6 (Bonus — Advanced): Thin Provisioning & LVM RAID

### Quick Learning

**Thin provisioning** (`lvcreate -T`) lets you allocate LVs that report a larger size than the physical space actually backing them — space is drawn from a shared thin pool only as data is actually written, similar to thin-provisioned LUNs you'd configure on a storage array. This is powerful but dangerous: if the thin pool itself fills up because *multiple* thin LVs are over-committing simultaneously, writes across *all* of them can start failing at once, with no per-LV warning. **LVM RAID** (`lvcreate --type raid1/raid5/raid6`) is the modern, actively-maintained replacement for old-style LVM mirroring, implemented via the kernel's `dm-raid` target — same caveat as Topic 5 applies: rarely needed when SAN/cloud storage already provides redundancy below LVM.

### Implementation (Learn by Applying)

**Scenario:** Build a thin pool, provision two thin LVs that together over-commit the pool, and observe what happens as they fill it — this is the exact failure mode you need to be able to describe from firsthand experience, not just theory.

```bash
# Create a thin pool with 5G of real backing space
lvcreate -L 5G -T vg_app/thinpool
lvs vg_app

# Provision TWO thin LVs, each claiming 10G — deliberately over-committing the 5G pool
lvcreate -V 10G -T vg_app/thinpool -n lv_thin1
lvcreate -V 10G -T vg_app/thinpool -n lv_thin2
lvs -o +lv_size,data_percent vg_app     # Both show 10G size, but the pool itself only has 5G real space

mkfs.xfs /dev/vg_app/lv_thin1
mkdir -p /thin1
mount /dev/vg_app/lv_thin1 /thin1

# Watch the pool fill as you write — this is the moment that matters
dd if=/dev/zero of=/thin1/fillfile bs=1M count=4000
lvs -o +data_percent vg_app/thinpool    # Check how close the POOL (not the LV) is to full
```

Try a quick LVM RAID1 for comparison of syntax and concept (skip if disk-constrained — reading the commands is enough here):
```bash
lvcreate --type raid1 -m1 -n lv_raid_test -L 2G vg_app
lvs -a -o +devices vg_app | grep raid    # Notice the extra "rimage"/"rmeta" sub-LVs LVM creates to manage the mirror
```

### Interview Questions

1. What actually happens to writes on Thin LV #1 if Thin LV #2 (sharing the same pool) fills the pool to 100% first? Is it graceful or does it fail hard?
2. How would you monitor a thin pool in production to get ahead of the "pool is full" scenario before it becomes an incident?
3. What's the relationship between thin provisioning at the LVM layer and thin provisioning your storage team might already be doing at the SAN/array layer — is stacking both a good idea, and why or why not?
4. When would LVM RAID actually be the right call, versus relying on redundancy already provided by underlying storage?
5. If you inherited a system using thin LVs and had no documentation, what commands would you run to quickly understand current over-commit ratio and remaining headroom?

---

**End of Day 1.** You should now be able to build an LVM stack from raw disks, grow it live with zero downtime, explain (and demonstrate) exactly where and why shrinking breaks down for XFS, reason correctly about Copy-on-Write snapshot sizing and invalidation, and speak concretely — from a real lab, not just theory — about disk-failure resilience and thin-pool over-commit risk.

Proceed to **Day 2 — Networking** next.
