# Day 1 — LVM & Storage Fundamentals (Full Study Edition, with Answers)

**Estimated time: 4–5 hours** (this edition includes worked answers — read the interview questions, attempt them yourself first, THEN check your reasoning against the answers. Don't just read the answers passively, or you'll recognize them in an interview without being able to produce them.)

This day covers the full storage lifecycle a product-company SysAdmin actually touches: LVM from raw disks through resilience, plus the modern RHEL-native layers (Stratis, VDO) and DM-Multipath — directly relevant given your SAN/Power Systems background. Each topic now opens with a **Quick Review** (a fast recap you can return to later without re-reading the full explanation), followed by the deeper **Quick Learning**, a hands-on **Implementation**, and **Interview Questions with detailed answers**.

---

## Topic 1: LVM Architecture — PV, VG, LV

### Quick Review
- **PV (Physical Volume)** — a raw disk/partition initialized for LVM use.
- **VG (Volume Group)** — a pool combining one or more PVs.
- **LV (Logical Volume)** — a carved-out slice of a VG, formatted and mounted like a virtual disk.
- Space inside a VG is divided into fixed-size **Physical Extents (PEs)**; an LV is just an ordered set of PE assignments — this is why resizing is cheap (reassigning pointers, not moving data).

### Quick Learning

LVM adds an abstraction layer between raw disks and filesystems so storage can be resized, moved, and pooled without caring about the underlying physical layout. The PE-mapping model is the mechanism behind almost every LVM superpower you'll use this week: growing, shrinking, snapshotting, and migrating live all boil down to extent bookkeeping. Understanding this at the mechanism level — not just "LVM lets you resize things" — is what separates a memorized answer from a real one in an interview.

**The LVM stack, top to bottom:**
```
        Application  ( /appdata )
              │
              ▼
     ┌──────────────────┐
     │  Filesystem (XFS)  │   <- what df -h actually reads
     └──────────────────┘
              │
              ▼
     ┌──────────────────────────────┐
     │   Logical Volume: lv_app       │   <- 15GB, made of PEs
     │   (an ordered list of PEs)     │
     └──────────────────────────────┘
              │  drawn from
              ▼
     ┌───────────────────────────────────────────┐
     │           Volume Group: vg_app               │
     │   (pooled Physical Extents from all PVs)     │
     └───────────────────────────────────────────┘
        │              │              │
        ▼              ▼              ▼
   ┌─────────┐   ┌─────────┐   ┌─────────┐
   │  PV       │   │  PV       │   │  PV       │
   │ /dev/sdb  │   │ /dev/sdc  │   │ /dev/sdd  │
   │  10GB     │   │  10GB     │   │  10GB     │
   └─────────┘   └─────────┘   └─────────┘
```
Every resize/snapshot/migrate operation you'll do this week is really just moving up and down this stack — reassigning which PEs an LV points to (LV/VG layer) and telling the filesystem to notice (filesystem layer). Keep this picture in your head; most "why didn't my resize work" bugs are really "I only touched one layer of this diagram."

### Implementation (Learn by Applying)

**Scenario:** You've been handed a fresh RHEL 9 VM with three unused 10GB disks (`/dev/sdb`, `/dev/sdc`, `/dev/sdd`). Build an LVM stack pooling all three into one VG, and carve out a 15GB LV for an application that will need to grow later.

```bash
lsblk -f
pvcreate /dev/sdb /dev/sdc /dev/sdd
pvs

vgcreate vg_app /dev/sdb /dev/sdc /dev/sdd
vgs -o +vg_free_count,vg_extent_count

lvcreate -n lv_app -L 15G vg_app
lvs -o +devices

mkfs.xfs /dev/vg_app/lv_app
mkdir -p /appdata
mount /dev/vg_app/lv_app /appdata
blkid /dev/vg_app/lv_app
echo "UUID=$(blkid -s UUID -o value /dev/vg_app/lv_app) /appdata xfs defaults,nofail 0 0" >> /etc/fstab
mount -a
```

Deliberately leave ~15GB unallocated in the VG rather than using `100%FREE` — Topic 2 uses that headroom.

### Interview Questions — with Answers

**1. Walk me through what actually happens at each layer — PV, VG, LV — when I run `df -h` and see 15GB available on `/appdata`. Where does that number actually come from?**

`df -h` reads filesystem-level metadata (from XFS's own superblock, in this case), which reports how much space the filesystem was told it has — that number was set when the LV was created at 15GB and the filesystem was built to fill it. That 15GB, in turn, is just an accounting entry in the VG: LVM allocated `15GB / PE_size` extents to `lv_app` out of the pool contributed by the three PVs. Nothing about `df -h`'s output is "live" from the disks — it's the filesystem trusting the size it was formatted with, which trusts the LV size, which trusts the VG's extent allocation. If any of those layers get out of sync (e.g., LV grown but filesystem not grown), `df -h` will report the *old*, wrong number until you fix the filesystem layer specifically.

**2. Why would a team choose LVM over just formatting a disk directly, especially in a cloud/VM environment where the disk itself can often already be resized at the hypervisor level?**

Even when the hypervisor/cloud provider can resize the underlying virtual disk, that only solves *one* dimension — you still need to grow the partition and filesystem on top, which LVM makes trivial and safe (`lvextend -r`) versus juggling raw partition tables live. More importantly, LVM gives you capabilities the hypervisor layer doesn't: pooling multiple disks into one flexible space, live snapshots for safe rollback points, and the ability to move data between physical devices (`pvmove`) without downtime — none of which "just resize the cloud disk" gives you. It also decouples your storage management workflow from whichever specific cloud/hypervisor you're on, which matters for consistency across hybrid or multi-cloud environments.

**3. If I run `lvcreate -l 100%FREE`, what specifically am I giving up compared to leaving unallocated space in the VG?**

You lose headroom for anything that needs *VG-level* free extents without adding a new disk: creating a new LV later, growing an existing LV without first attaching new storage, or taking a properly-sized snapshot (which draws its overflow space from the VG's free extents). Using `100%FREE` isn't wrong for a single, permanent, non-growing LV, but for most production LVs, leaving deliberate slack is the safer default because it gives you emergency room to extend or snapshot without a mid-incident scramble to attach new storage first.

**4. You inherit a server where `vgs` shows a VG with 0% free space, but the LV mounted from it shows only 40% used in `df -h`. How do you explain that gap, and what would you check first?**

This is the classic LVM/filesystem-layer confusion: `vgs` reporting 0% free means every extent in the VG is *allocated* to some LV (or LVs) — it says nothing about how full the *filesystem inside* those LVs actually is. The LV itself was allocated its full size at the VG layer, but the filesystem on top is only 40% utilized in terms of actual data written. I'd check `lvs` to confirm how many LVs exist and their sizes (maybe there are other LVs consuming the rest of the VG's extents that aren't `/appdata` at all), then `df -h` across all mounted filesystems on this VG to build the full picture before assuming anything is wrong.

**5. Can you mix PVs with different underlying block sizes or from entirely different storage backends (e.g., one local disk, one SAN LUN) in the same VG? What are the practical risks of doing so?**

Technically LVM allows it — it doesn't enforce homogeneous backing storage. The practical risk is performance inconsistency: if an LV's extents get striped or allocated across both a fast local NVMe disk and a slower SAN LUN, the LV's overall performance characteristics become unpredictable and hard to reason about, and a failure or latency spike on the slower device can bottleneck the whole LV even if most of its data lives on the fast device. In production, best practice is keeping VGs backed by storage of consistent performance/reliability characteristics, and using separate VGs (or explicit PV targeting with `lvcreate` device arguments) when mixing tiers is unavoidable.

---

## Topic 2: Extending Storage Without Downtime

### Quick Review
- Extending is a **two-layer** operation: grow the **LV** (claim more VG extents), then grow the **filesystem** on top.
- `lvextend -r` does both atomically in one command.
- XFS and ext4 can both grow while mounted and live.
- If the VG itself has no free extents, you need a new PV (`pvcreate` + `vgextend`) before you can extend the LV further.

### Quick Learning

This is the operation you'll perform most often as a working SysAdmin, and it's also the most common source of a specific, avoidable mistake: extending only the LV and forgetting the filesystem layer, leaving the application still starved for space despite `lvextend` reporting success. Internalizing that these are two distinct operations — even when you use the combined `-r` flag — is what lets you debug it correctly when someone else made that mistake.

**The mistake, visually:**
```
  lvextend -L +10G   (LV layer only)              lvextend -r -L +10G   (both layers)
  ─────────────────────────────────              ─────────────────────────────────
  LV size:  15G ──▶ 25G  ✔ grown                 LV size:  15G ──▶ 25G  ✔ grown
  FS size:  15G ──▶ 15G  ✘ UNCHANGED              FS size:  15G ──▶ 25G  ✔ grown

  df -h still shows 15G total  ✘                 df -h shows 25G total  ✔
  (fix: xfs_growfs / resize2fs, run separately)
```

### Implementation (Learn by Applying)

**Scenario:** The application team on `/appdata` needs 10 more GB *today*, zero downtime, using the headroom left in Topic 1.

```bash
vgs vg_app
lvextend -r -L +10G /dev/vg_app/lv_app
lvs vg_app/lv_app
df -hT /appdata
xfs_info /appdata | head -5
```

Now simulate the VG itself running out of headroom — a "SAN team just presented a new LUN" scenario:

```bash
pvcreate /dev/sde
vgextend vg_app /dev/sde
vgs vg_app
lvextend -r -L +5G /dev/vg_app/lv_app
lvs -o +devices /dev/vg_app/lv_app
```

### Interview Questions — with Answers

**1. A junior admin runs `lvextend -L +10G /dev/vg_app/lv_app` (no `-r`) and says "it didn't work, `df -h` still shows the old size." What happened, and what two commands fix it?**

The LV itself *did* grow — `lvs` would confirm the new, larger size — but the filesystem sitting on top of it was never told to use the newly available space, so `df -h`, which reads filesystem metadata, still reports the old size. The fix is running the filesystem-specific grow command against the now-larger LV: `xfs_growfs /appdata` for XFS (must be run against the mount point, not the device) or `resize2fs /dev/vg_app/lv_app` for ext4 (can target the device directly). Confirm with `df -h` afterward.

**2. Can you grow an XFS filesystem while mounted and actively written to? What about ext4? Any circumstances requiring unmount first?**

Both XFS and ext4 support online (mounted, live) growth — this is one of the most useful properties of modern Linux filesystems for zero-downtime operations. The circumstance requiring unmount is the *reverse* operation, shrinking: ext4 requires unmounting to shrink, and XFS can't shrink at all regardless of mount state. So growing is always safe live; shrinking is a different story entirely (covered in Topic 3).

**3. The VG has zero free extents and no new physical disk is available. What are your actual options, ranked by how disruptive they are?**

Least disruptive: check if another LV in the same VG has unused allocated space that can be shrunk (`lvreduce`) to free extents for reallocation — only viable if that LV is ext4 and has genuine slack, and requires a maintenance window for that other LV. Next: request new storage (a new virtual disk in a VM, or a new LUN from the SAN team) and `vgextend` once available — this is the standard path and involves no downtime for the LV needing more space, just lead time waiting on provisioning. Most disruptive: migrate the LV's data elsewhere (larger volume, different VG) via backup/restore or `pvmove` to a bigger replacement disk, which typically does involve a maintenance window. In practice, "request more storage" is almost always the actual answer — the other options are fallbacks for when that's not immediately possible.

**4. Walk me through your debugging steps: a user says "LVM won't extend" — `lvextend` hangs or errors. What do you check, in what order?**

First, check the actual error message rather than assuming — `lvextend` errors are usually explicit (e.g., "insufficient free extents"). Check `vgs` to confirm the VG actually has free space to draw from; if not, that's the whole problem, and no amount of retrying `lvextend` fixes it. If there IS free space but it's failing, check whether the LV is part of a more complex setup (thin pool, mirror, RAID) where `lvextend` semantics differ — thin-provisioned LVs, for instance, extend against pool space, not raw VG free extents. If it's "hanging" rather than erroring, check for a stuck LVM lock (`dmsetup info`, `lvs` from another terminal to see if metadata is even readable) or an underlying disk I/O issue on one of the VG's PVs — a slow/failing PV can make LVM metadata operations hang rather than fail cleanly.

**5. If an LV spans multiple PVs after growth, does that create any performance implications, positive or negative, versus an LV living entirely on one PV?**

It can go either way depending on allocation policy and the underlying devices. If LVM used a **linear** allocation (default), the LV just continues sequentially onto the new PV — no meaningful performance change, but I/O to different parts of the LV now hits different physical disks, which is fine for most workloads and can even help if the PVs are independent devices with separate I/O paths. If instead the extents were striped across PVs (explicit `-i`/`-I` striping), you can get real throughput gains from parallel I/O across devices — but only if all PVs involved have comparable performance; mixing a fast and slow disk in a striped LV drags overall performance down to the slowest device. The practical answer: check `lvs -o +devices` and `lvs -o +stripes` to know which situation you're actually in before assuming either a benefit or a penalty.

---

## Topic 3: Shrinking & Reducing — Where It Breaks Down

### Quick Review
- Shrink order is the **reverse** of extending: filesystem first, then the LV.
- **XFS cannot shrink at all** — no `xfs_shrinkfs` exists, by design.
- **ext4 can shrink**, but only while **unmounted**.
- Always run a filesystem check (`e2fsck -f`) before shrinking ext4.

### Quick Learning

Shrinking is the operation most admins have theoretical knowledge of but little hands-on experience with, precisely because it's riskier and less frequently needed than growing. The core danger: if you shrink the LV before the filesystem, you can truncate data blocks the filesystem still believes it owns, leading to corruption. XFS's designers made a deliberate architectural choice to never support shrinking at all, trading that flexibility for simplicity and reliability elsewhere in the filesystem's design — know this as a design tradeoff, not a limitation to be embarrassed about when asked.

**Correct order vs. the corrupting order:**
```
  CORRECT (ext4 only)                    WRONG — never do this
  ────────────────────                   ────────────────────
  1. umount                              1. lvreduce  (LV shrunk first)
  2. e2fsck -f                              └─▶ blocks the FS still thinks
  3. resize2fs <new size>                        it owns are now GONE
       (FS now fits new size)                └─▶ corruption / unmountable FS
  4. lvreduce -L <new size>
       (LV shrunk to match, safe)         2. resize2fs  (too late — FS already
  5. mount                                    lost blocks it needed)

  XFS: step 3 has no equivalent command — there is no xfs_shrinkfs, period.
```

### Implementation (Learn by Applying)

**Scenario:** An over-provisioned 20GB ext4 logging volume needs to give back 10GB, with a short maintenance window allowed.

```bash
lvcreate -n lv_logs -L 20G vg_app
mkfs.ext4 /dev/vg_app/lv_logs
mkdir -p /logs
mount /dev/vg_app/lv_logs /logs

umount /logs
e2fsck -f /dev/vg_app/lv_logs
resize2fs /dev/vg_app/lv_logs 9G
lvreduce -L 10G /dev/vg_app/lv_logs
mount /dev/vg_app/lv_logs /logs
df -hT /logs
```

Prove XFS's limitation to yourself:
```bash
lvcreate -n lv_xfs_test -L 10G vg_app
mkfs.xfs /dev/vg_app/lv_xfs_test
mount /dev/vg_app/lv_xfs_test /mnt
umount /mnt
xfs_growfs -n /mnt 2>&1 || true    # There is no xfs_shrinkfs
```

### Interview Questions — with Answers

**1. Why can't XFS shrink, architecturally? At minimum, what's your practical workaround when the business demands it?**

XFS's on-disk layout (allocation groups, B+tree-based free space and inode management) was designed around the assumption of a fixed or growing filesystem, prioritizing scalability and performance for large filesystems over the flexibility to shrink. Red Hat and the XFS maintainers have never implemented shrink support, treating it as a deliberate scope decision rather than a missing feature they intend to add. The practical workaround: create a new, smaller LV, format it XFS, copy the data across (`rsync` or similar) during a maintenance window, then swap the mount point/fstab entry to point at the new smaller volume and remove the old one.

**2. What happens, mechanically, if someone shrinks the LV *before* the filesystem? Why is order non-negotiable for ext4?**

If the LV is shrunk first, the block device the filesystem sits on suddenly becomes smaller than what the filesystem's own metadata says it should be — any filesystem structures (inodes, data blocks, journal) that lived in the now-removed extents are simply gone, out from under the filesystem, with no warning. This produces filesystem corruption, potentially including loss of the journal or critical metadata structures, and can leave the filesystem completely unmountable. Shrinking the filesystem first ensures ext4 relocates any data/metadata out of the region about to be removed and updates its own bookkeeping to reflect the new, smaller size — only after that is it safe to actually remove the underlying blocks via `lvreduce`.

**3. When would you actually recommend shrinking in production versus leaving over-provisioned space alone?**

Shrinking is rarely worth the operational risk unless there's a concrete, valuable reason to reclaim the space — for example, the VG is genuinely constrained and other LVs urgently need the extents, or you're consolidating storage during a broader migration anyway. If the "waste" is just unused capacity sitting in an otherwise healthy, adequately-sized VG with no competing demand for that space, I'd generally recommend leaving it alone — the risk of a shrink operation going wrong (even with correct procedure) usually outweighs the benefit of reclaiming space nobody currently needs.

**4. A colleague wants to shrink a live, actively-written ext4 volume with no maintenance window. What do you tell them?**

I'd tell them it's not possible safely — ext4 shrinking requires the filesystem to be unmounted, full stop, so "no maintenance window" and "shrink this live" are mutually exclusive requirements. Their real options are: accept a short maintenance window (even brief downtime is required), or achieve the same practical goal a different way — e.g., migrate the actively-written workload to a new, appropriately-sized volume via application-level replication/cutover rather than a filesystem-level shrink, which can sometimes be done with less disruption if the application supports it.

**5. Walk me through your actual migration approach to "shrink" an XFS volume, including how you'd minimize downtime.**

First, provision the new, smaller target LV and format it XFS while the old volume is still live and serving traffic — no downtime yet. Then do an initial `rsync` (or equivalent) copy of the data while the source is still active, which captures the bulk of the data without any outage. Schedule a short maintenance window for the final cutover: stop the application/unmount the old volume, run a final incremental `rsync` to catch anything that changed since the first pass (fast, since most data already copied), then remount the application's data path at the new volume's mount point (or swap fstab/symlinks) and start the application back up. This "bulk copy live, then short final sync during a brief window" pattern minimizes actual downtime to roughly the time needed for the delta sync plus the app restart, rather than the time needed to copy the entire dataset.

---

## Topic 4: Snapshots & Copy-on-Write

### Quick Review
- LVM snapshots use **Copy-on-Write (COW)**: a block is copied to the snapshot only the *first time* it changes on the origin after the snapshot was taken.
- Snapshot size = overflow room for expected **writes during its life**, not a copy of the whole volume.
- Exceeding that overflow **invalidates** the snapshot silently — it becomes unusable.
- `lvconvert --merge` rolls the origin back to the snapshot's point in time.

### Quick Learning

The single most common real-world snapshot mistake is treating the `-L` size as "how big a copy of my data will be" rather than "how much change I expect during this snapshot's lifetime." A 500GB volume with almost no write activity during a snapshot's life needs a tiny snapshot; a 50GB volume under heavy write load during the snapshot window might need a surprisingly large one. Getting this wrong doesn't cause an error at snapshot-creation time — it causes a silent failure later, exactly when you need the rollback to work.

**Copy-on-Write, step by step:**
```
  Time 0: snapshot taken                  Time 1: origin block A is overwritten
  ┌──────────────┐                        ┌──────────────┐
  │  Origin LV     │                        │  Origin LV     │
  │  [A][B][C][D]  │                        │  [A'][B][C][D] │  <- new data written
  └──────────────┘                        └──────────────┘
  ┌──────────────┐                        ┌──────────────┐
  │  Snapshot      │                        │  Snapshot      │
  │  (empty, just   │   BEFORE the write    │  [A ] ← ← ← ←  │  old "A" copied here
  │   overflow room)│   to A completes,     │  (only A used   │  FIRST, then the write
  └──────────────┘   old A is copied ──▶  │   space so far) │  to origin is allowed
                        into the snapshot   └──────────────┘

  Block B, C, D never changed ⇒ never copied ⇒ snapshot only ever
  pays storage cost for blocks that actually change on the origin.
```
If cumulative changed-block volume exceeds the snapshot's `-L` size before you remove/merge it, the snapshot is **invalidated** — silently unusable, even though the origin is completely fine.

### Implementation (Learn by Applying)

```bash
lvcreate -s -n lv_app_snap -L 500M /dev/vg_app/lv_app
lvs -o +snap_percent vg_app

dd if=/dev/zero of=/appdata/testfile bs=1M count=100
lvs -o +snap_percent vg_app

umount /appdata
lvconvert --merge vg_app/lv_app_snap
mount /dev/vg_app/lv_app /appdata     # Confirm testfile is gone
```

Force an invalidation:
```bash
lvcreate -s -n lv_app_snap2 -L 100M /dev/vg_app/lv_app
dd if=/dev/zero of=/appdata/bigfile bs=1M count=500
lvs -o +snap_percent vg_app
lvdisplay /dev/vg_app/lv_app_snap2    # Check "Status" — this is what invalidation looks like
lvremove vg_app/lv_app_snap2
```

### Interview Questions — with Answers

**1. Explain Copy-on-Write in your own words — when a write hits the origin, what's copied, and when?**

When the origin volume receives a write to a block that hasn't been touched since the snapshot was created, LVM intercepts that write, first copies the *original, pre-write* content of that block into the snapshot's reserved space, and only then allows the actual new write to proceed on the origin. Subsequent writes to that *same* block don't trigger another copy — the "before" state is already preserved. This means the snapshot's storage consumption grows in proportion to how much *unique* data changes on the origin, not how much total write traffic occurs — a block overwritten 100 times still only costs one COW copy.

**2. How do you size a snapshot correctly for a production maintenance window?**

Estimate based on the *write volume* expected during the snapshot's planned lifetime, not the volume's total size — for a database undergoing a schema migration, that might mean estimating how much data the migration process itself rewrites, plus any concurrent application writes if the app isn't fully quiesced. In practice, I'd look at historical write throughput for that volume (via `iostat`/`sar` history) over a comparable time window, multiply by the planned snapshot duration with a safety margin (2-3x is a reasonable rule of thumb), and monitor `lvs -o +snap_percent` throughout the maintenance window regardless, so an underestimate doesn't turn into a silent surprise.

**3. A snapshot shows 100% full. Is the origin's data at risk? Is the snapshot still usable for rollback?**

The origin's live data is *not* at risk — the origin volume continues operating normally and writes keep succeeding, because once the snapshot overflows, LVM simply stops maintaining it rather than blocking origin writes. However, the snapshot itself becomes **invalid** at that point and is no longer usable for rollback — you've lost your restore point going forward, even though the origin itself is fine. This is exactly why monitoring `snap_percent` during the window matters: catching it approaching full lets you either extend the snapshot's size live (`lvextend` works on snapshots too) or abandon and recreate it, before it silently becomes useless.

**4. What's the operational difference between an LVM snapshot as a rollback mechanism versus as a backup source?**

As a rollback mechanism, the snapshot stays attached to the origin and is either merged back (`lvconvert --merge`) to undo changes, or discarded if the risky operation succeeded — it's inherently temporary and lives on the same storage as the origin, so it doesn't protect against a full disk/array failure. As a backup source, you'd typically mount the snapshot read-only at a separate mount point and copy its point-in-time data off to genuinely separate storage (tape, object storage, a different array) — the snapshot itself is just a consistent, unchanging view to read from during the copy, not the backup itself. Confusing the two is a real risk: relying on a same-disk snapshot as your only "backup" doesn't protect you if that physical disk/array fails entirely.

**5. Walk me through your monitoring approach so nobody gets paged because a snapshot silently invalidated overnight.**

I'd set up active alerting on `lvs -o +snap_percent` (or the equivalent metric via whatever monitoring agent is in place) with a threshold well before 100% — e.g., alert at 70-80% fill so there's time to react before invalidation, not after. For any snapshot taken as part of a scheduled/automated job (like a pre-patch backup snapshot), I'd build the fill-percentage check directly into the automation itself, so the job fails loudly and visibly if the snapshot is at risk, rather than relying purely on a separate monitoring system to catch it. Finally, I'd make sure snapshots have a defined lifecycle — an expectation for how long they should exist — so a forgotten snapshot from weeks ago isn't silently consuming VG space and isn't the thing that finally tips into invalidation during an unrelated write spike.

---

## Topic 5: Resilience — Surviving a Disk Failure

### Quick Review
- LVM-level mirroring/RAID protects against single disk failure but is often redundant when SAN/cloud storage already provides redundancy below LVM.
- Design VGs so a single disk loss doesn't take out every LV.
- `vgs`/`vgdisplay` reveal a missing PV via the Attr "p" flag or a current-vs-active PV count mismatch.
- Proactive monitoring (SMART data) catches failing disks before the kernel starts throwing I/O errors.

### Quick Learning

"Resilience" is a word candidates use loosely, and interviewers use questions about it to test whether you actually understand *where* redundancy lives in your stack. If your storage is SAN-backed with array-level RAID and multipath, adding LVM mirroring on top is usually redundant complexity with a real performance cost and no meaningful benefit — the actual skill being tested is knowing *which layer* is responsible for what, not defaulting to "more redundancy is always better."

**Blast radius — why VG layout matters:**
```
  BAD: one giant VG                       BETTER: isolated VGs per tier
  ─────────────────────                   ─────────────────────────────
  vg_all                                   vg_critical          vg_scratch
  ├─ PV: sdb  ├─ PV: sdc                   ├─ PV: sdb            ├─ PV: sdd
  ├─ lv_db      (spans sdb+sdc)            └─ lv_db               └─ lv_tmpfiles
  ├─ lv_logs    (spans sdb+sdc)                (isolated —            (isolated —
  └─ lv_tmp      (spans sdb+sdc)                sdc failure           sdd failure
                                                 doesn't touch it)     doesn't touch db)
  sdc fails ⇒ lv_db, lv_logs, AND lv_tmp
  can ALL be affected, even though only
  one physical disk actually died.
```

### Implementation (Learn by Applying)

```bash
pvs
pvs -a
```

After detaching a disk belonging to `vg_app` at the hypervisor level:
```bash
vgscan
pvscan
vgs vg_app                    # Look for "p" (partial) in the Attr column
vgdisplay vg_app | grep -i "cur pv\|act pv"
```

Recovery approach:
```bash
pvcreate /dev/sdX_new
vgextend vg_app /dev/sdX_new
vgreduce --removemissing vg_app
```

Proactive monitoring:
```bash
smartctl -a /dev/sdb | grep -E "Reallocated_Sector_Ct|Current_Pending_Sector|SMART overall-health"
```

### Interview Questions — with Answers

**1. In a SAN or cloud-block-storage environment, why might a team deliberately avoid LVM-level mirroring even though it's available?**

Because the redundancy already exists one layer down — SAN arrays typically implement RAID internally across physical disks, and cloud block storage (like EBS) replicates data across multiple physical devices/availability zones as part of the service itself. Adding LVM mirroring on top means every write gets duplicated *again* at the OS layer, doubling I/O and consuming double the logical capacity, for protection against a failure mode (a single physical disk dying) that the layer below has already solved. The only time LVM-level redundancy earns its cost is when the underlying storage genuinely doesn't provide its own redundancy — e.g., truly raw, unprotected local disks with no array or cloud-level replication underneath.

**2. A VG shows one PV missing. How do you determine which LVs are affected versus untouched?**

Run `lvs -a -o +devices` (or `pvdisplay -m` on the healthy remaining PVs, and check `pvdisplay -m` output that referenced the missing one before it went missing, if you have prior documentation) to see exactly which LVs have extents allocated on the missing PV versus which LVs live entirely on the surviving PVs. Any LV with *zero* extents on the failed PV is completely unaffected and continues operating normally; any LV with extents on the failed PV either has degraded/missing data (if not mirrored) or is running in a degraded-but-functional mirrored state (if it was mirrored). Cross-referencing `lvs -a -o +devices` against the known failed PV's device name is the direct way to build this list.

**3. What's the difference between `vgreduce --removemissing` with and without `--force`, and why does it matter for data loss?**

`vgreduce --removemissing` without `--force` is more conservative — it will refuse to remove the missing PV's references if doing so would affect an LV that has extents (partially or fully) on that PV, protecting you from accidentally destroying an LV's remaining metadata consistency. Adding `--force` overrides that safety check and proceeds regardless, which can result in genuinely broken/incomplete LVs (any LV that had data uniquely on the missing PV loses that data permanently, and LVM metadata is updated to reflect the LV as smaller/broken). In practice, I'd only use `--force` after confirming (via the devices check above) exactly which LVs are impacted and accepting that any non-mirrored LV with extents on the dead PV has already lost that data regardless of the flag — `--force` just makes LVM stop treating that as a blocking condition.

**4. How would you design a VG layout from the start to minimize blast radius from a single disk failure?**

I'd avoid a single "giant VG with everything in it" pattern for critical, differently-tiered workloads — instead grouping LVs by criticality/blast-radius tolerance into separate VGs, each backed by its own set of PVs, so a disk failure affecting one VG's PVs doesn't cascade into unrelated LVs living in a different VG on different physical disks. For workloads where the underlying storage doesn't already provide redundancy, I'd also consider explicit LVM mirroring/RAID for the specific LVs where data loss is unacceptable, rather than applying it uniformly (which wastes capacity on data that could tolerate reprovisioning from a backup instead). The overarching principle: match the redundancy/isolation strategy to the actual criticality of the data, rather than a one-size-fits-all layout.

**5. Describe your monitoring/alerting approach for storage health across the full stack — what pages you at night versus what just emails you?**

At the disk/hardware layer, SMART attributes like `Reallocated_Sector_Ct` and `Current_Pending_Sector` trending upward, or a SAN array reporting a degraded RAID group, would page immediately — these are early warnings of imminent failure where proactive replacement avoids an outage. At the LVM/filesystem layer, a VG showing a missing/partial PV, or filesystem usage crossing a high threshold (e.g., 85-90%) with a trend suggesting it'll hit 100% within hours, would also page. Lower-urgency signals — filesystem usage crossing a lower threshold (e.g., 70%) with a slow growth trend, or a snapshot approaching its fill percentage days before it's due for removal anyway — would go to email/ticket rather than an immediate page, since there's comfortable lead time to act during business hours.

---

## Topic 6: Thin Provisioning & LVM RAID

### Quick Review
- Thin LVs (`lvcreate -T`) report more logical space than the pool physically has; space is consumed only as data is actually written.
- Multiple thin LVs sharing one pool can **collectively** exhaust it, failing writes across all of them simultaneously.
- LVM RAID (`--type raid1/5/6`) is the modern `dm-raid`-based redundancy option, generally unnecessary when SAN/cloud storage already provides redundancy.

### Quick Learning

Thin provisioning trades guaranteed capacity for flexibility — you can over-commit generously based on the assumption that not every LV will actually use its full logical allocation simultaneously. That assumption holds right up until it doesn't, and when it fails, it fails for *every* thin LV in the pool at once, not gracefully or individually — which is exactly the kind of failure mode interviewers want to hear you can reason about and monitor for proactively.

**Over-commitment, visually:**
```
  What consumers SEE (logical)             What actually EXISTS (physical pool)
  ───────────────────────────              ─────────────────────────────────
  lv_thin1:  ▓▓▓▓▓▓▓▓▓▓ 10G                ┌────────────────────────┐
  lv_thin2:  ▓▓▓▓▓▓▓▓▓▓ 10G                │   thinpool: 5G actual   │
  ───────────────────────────              │   physical space total  │
  Logical total: 20G "available"           └────────────────────────┘
                                              ▲
  Both LVs writing heavily at once ──────────┘
  drains the SAME 5G pool. Whichever write
  arrives after the pool hits 100% fails —
  even if it's on the LV that used LESS space.
```

### Implementation (Learn by Applying)

```bash
lvcreate -L 5G -T vg_app/thinpool
lvs vg_app

lvcreate -V 10G -T vg_app/thinpool -n lv_thin1
lvcreate -V 10G -T vg_app/thinpool -n lv_thin2
lvs -o +lv_size,data_percent vg_app

mkfs.xfs /dev/vg_app/lv_thin1
mkdir -p /thin1
mount /dev/vg_app/lv_thin1 /thin1
dd if=/dev/zero of=/thin1/fillfile bs=1M count=4000
lvs -o +data_percent vg_app/thinpool
```

```bash
lvcreate --type raid1 -m1 -n lv_raid_test -L 2G vg_app
lvs -a -o +devices vg_app | grep raid
```

### Interview Questions — with Answers

**1. What happens to writes on Thin LV #1 if Thin LV #2 (same pool) fills the pool first? Graceful or hard failure?**

It's a hard failure, and it's not per-LV isolated — once the shared pool runs out of physical space, *any* thin LV in that pool attempting to write new (previously unallocated) blocks will fail, regardless of which LV's activity actually caused the pool to fill. This is the key danger of thin provisioning: Thin LV #1 can be a well-behaved, low-growth volume and still get write failures caused entirely by Thin LV #2's runaway growth, because they share the same underlying physical capacity with no per-LV hard ceiling by default (unless you've explicitly configured pool-level thresholds/policies to prevent this).

**2. How would you monitor a thin pool in production to get ahead of "pool is full"?**

I'd monitor the pool's `data_percent` (via `lvs -o +data_percent` or the equivalent monitoring-agent metric) with alerting well before it approaches 100% — the pool filling is a shared-resource event affecting every thin LV riding on it, so the alert threshold should be more conservative than a typical single-filesystem-full alert, e.g., paging at 75-80% given how fast over-committed pools can accelerate toward full. I'd also track the over-commit ratio itself (sum of logical sizes of all thin LVs versus actual pool physical size) as a standing metric, since a pool that's already heavily over-committed needs tighter monitoring than one with modest over-commitment.

**3. Is stacking thin provisioning at both LVM and SAN layers a good idea? Why or why not?**

Generally not — stacking two independent thin-provisioning/over-commit layers compounds the risk without adding real benefit, because you now have two separate "we assumed not everyone will use their full allocation" bets stacked on top of each other, and either layer running out independently causes a failure. It also makes capacity planning and troubleshooting significantly harder, since "why did this write fail" now requires checking over-commit state at two different layers instead of one. If the SAN already thin-provisions LUNs, I'd generally present those LUNs to LVM as regular (thick) PVs and skip LVM-level thin provisioning on top, letting one layer own that responsibility clearly.

**4. When would LVM RAID actually be the right call versus relying on underlying storage redundancy?**

LVM RAID earns its place when the underlying storage genuinely doesn't provide redundancy on its own — for example, a set of raw local disks in a bare-metal server or a budget VM environment with no array-level RAID and no cloud-managed replication underneath. It can also make sense in edge cases where you specifically want OS-visible, OS-managed redundancy for operational/tooling reasons (e.g., a team's existing monitoring/automation is built entirely around LVM RAID state rather than SAN-side alerting). Outside those cases — any environment where the SAN or cloud provider already handles physical redundancy — LVM RAID is usually unnecessary overhead.

**5. Inheriting an undocumented thin-provisioned system — what commands quickly reveal current over-commit ratio and headroom?**

`lvs -a -o +lv_size,data_percent,pool_lv` across the VG shows every thin LV's logical size, how full the *pool itself* actually is, and which pool each LV belongs to — from there, summing the logical sizes of thin LVs sharing a pool and comparing against the pool's actual physical size (`lvs` on the pool LV itself, or `vgs`/`lvs -o +lv_size` on the thinpool entry) gives the over-commit ratio directly. `lvs -o +data_percent` on the pool specifically is the single most important number to check first — it tells you how close to actual exhaustion the shared physical backing store is, which matters more immediately than the theoretical over-commit ratio.

---

## Topic 7: Stratis — RHEL's Next-Generation Storage Management

### Quick Review
- Stratis orchestrates existing components (device-mapper, XFS) via a daemon (`stratisd`) and CLI (`stratis`) — it is not a new filesystem itself.
- Workflow: `stratis pool create` → `stratis filesystem create` — no separate PV/VG/LV steps.
- All Stratis filesystems are thin-provisioned by default.
- Requires `x-systemd.requires=stratisd.service` in fstab, since mounting depends on the daemon being up.

### Quick Learning

Stratis exists to reduce the operational complexity of the classic PV→VG→LV→filesystem chain into two conceptual steps (pool, filesystem), aimed at closing the usability gap between Linux storage management and systems like ZFS. The tradeoff for that simplicity is losing some of LVM's granular manual control, and gaining a dependency on a userspace daemon (`stratisd`) that classic LVM doesn't have — the kernel can activate LVM volumes without any daemon running, but Stratis filesystems genuinely need `stratisd` alive to mount.

**Stratis vs. LVM, side by side:**
```
        LVM (manual, 4 steps)                 Stratis (orchestrated, 2 steps)
        ──────────────────────                ────────────────────────────
        pvcreate   (per disk)                  ┌─────────────────────────┐
              │                                 │        stratisd            │  <- userspace daemon,
              ▼                                 │   (must be running to     │     MUST be up before
        vgcreate   (pool disks)                 │      mount filesystems)   │     mount can succeed
              │                                 └─────────────────────────┘
              ▼                                            │  orchestrates
        lvcreate   (carve LV)                               ▼
              │                                 device-mapper thin pool + XFS
              ▼                                            │
        mkfs.xfs   (format)                                ▼
              │                                 stratis pool create  (1 step,
              ▼                                 pools raw disks directly)
        mount                                               │
                                                              ▼
        (kernel activates LVM,                  stratis filesystem create
         no daemon needed)                       (1 step, auto thin-provisioned)
                                                              │
                                                              ▼
                                                 mount  (needs stratisd alive —
                                                  fstab: x-systemd.requires=
                                                  stratisd.service)
```

### Implementation (Learn by Applying)

**Scenario:** A new team wants their storage to "just work" with automatic thin provisioning and simple snapshotting, without managing LVM layers by hand.

```bash
dnf install -y stratisd stratis-cli
systemctl enable --now stratisd

stratis pool create pool1 /dev/sdf /dev/sdg
stratis pool list
stratis blockdev list pool1

stratis filesystem create pool1 fs1
mkdir -p /stratis-data
mount /dev/stratis/pool1/fs1 /stratis-data
stratis filesystem list

blkid /dev/stratis/pool1/fs1
echo "UUID=$(blkid -s UUID -o value /dev/stratis/pool1/fs1) /stratis-data xfs defaults,x-systemd.requires=stratisd.service 0 0" >> /etc/fstab
```

```bash
stratis pool add-data pool1 /dev/sdh
stratis pool list

stratis filesystem snapshot pool1 fs1 fs1_snap
stratis filesystem list pool1
```

### Interview Questions — with Answers

**1. Conceptually, what is Stratis actually built on top of — is it a new filesystem, or something else? Why does that distinction matter when you're troubleshooting it?**

Stratis is an orchestration layer, not a new on-disk filesystem format — underneath, it's coordinating device-mapper constructs (thin pools, similar to LVM's own thin provisioning) with XFS as the actual filesystem living on top. This distinction matters for troubleshooting because when something goes wrong, the underlying tools you already know (device-mapper commands, `xfs_info`, `xfs_repair`) are still relevant and available — you're not dealing with an entirely foreign filesystem implementation with its own unfamiliar toolset, you're dealing with familiar components wired together by `stratisd`.

**2. Compare the LVM workflow (pvcreate → vgcreate → lvcreate → mkfs) to the Stratis workflow (pool create → filesystem create). What operational complexity does Stratis remove, and what control do you lose in exchange?**

Stratis collapses four manual, separately-verified steps into two, and handles thin provisioning and filesystem formatting automatically rather than requiring you to explicitly choose and configure them — for a team that just wants "storage that grows without babysitting," that's a real reduction in operational overhead and room for mistakes (like forgetting `-r` on an extend). What you lose is granular control: you can't easily choose "this LV is XFS, this one is ext4," you can't do explicit LVM-style striping/mirroring configuration the same way, and because Stratis pools are always thin-provisioned, you don't have the option of "thick" guaranteed-capacity volumes the way plain LVM offers by default.

**3. Why do Stratis filesystems need a special fstab option that plain LVM-backed filesystems don't? What actually breaks if you forget it?**

Classic LVM volumes are activated by the kernel's device-mapper subsystem directly during early boot, before systemd starts bringing up regular userspace services — no separate daemon needs to be running for an LVM LV to be available to mount. Stratis, by contrast, relies on `stratisd`, a userspace daemon, to assemble and expose its pools/filesystems as mountable devices — if systemd tries to mount a Stratis filesystem before `stratisd` has started, the mount will fail because the device simply doesn't exist yet from the kernel's point of view. The `x-systemd.requires=stratisd.service` fstab option tells systemd to order the mount attempt after `stratisd` is confirmed up; forgetting it produces intermittent or consistent mount failures at boot, especially on systems where boot timing/ordering happens to put the mount attempt before the daemon is ready.

**4. All Stratis filesystems in a pool are thinly provisioned by default. What's the operational risk of that, and how would you monitor for it?**

The same fundamental risk as LVM thin pools: multiple filesystems sharing a pool's physical backing storage can collectively exhaust it, causing write failures across filesystems that individually look fine. I'd monitor pool-level physical space consumption directly via `stratis pool list` (which reports total/used physical space per pool) with alerting well before the pool approaches full, the same conservative-threshold approach as LVM thin pools, since the failure mode and urgency are essentially identical.

**5. Would you recommend Stratis over traditional LVM for a new production deployment today? What would make you choose one over the other?**

I'd lean toward LVM for environments needing fine-grained control, mixed filesystem types, explicit non-thin allocation, or where the team already has deep LVM operational muscle memory and tooling built around it — which describes most established enterprise environments, including ones with a SAN/mainframe-adjacent background like mine. I'd consider Stratis for newer, simpler deployments genuinely valuing operational simplicity over granular control — for instance, a small team managing many similar servers where reducing the chance of manual LVM mistakes outweighs the loss of fine-grained tuning options. In practice, I'd also factor in organizational familiarity: introducing Stratis into a team with zero prior exposure carries its own risk, even if it's technically simpler, because troubleshooting an unfamiliar tool under incident pressure is its own cost.

---

## Topic 8: VDO — Virtual Data Optimizer (Deduplication & Compression)

### Quick Review
- VDO sits below a filesystem (or below LVM) and provides inline deduplication and compression.
- Presents a **logical size** to consumers that can exceed the **physical size** actually backing it — risk depends on real-world data reduction ratios holding up.
- Best suited to highly redundant data (backups, VM images); poor fit for already-compressed or unique/random data.
- `vdostats` is the tool for checking real physical consumption and savings ratio.

### Quick Learning

VDO's value proposition is straightforward — get more usable capacity out of the same physical disks for data that compresses or deduplicates well — but its risk profile deserves the same respect as thin provisioning's, because the "extra" capacity it presents is entirely contingent on an assumption (data reducibility) that can be wrong, unlike LVM thin provisioning's more predictable "not everyone uses their full allocation simultaneously" assumption.

**Where the "extra" space actually comes from:**
```
  Applications write:  5 x 1GB "identical-ish" backup copies  = 5GB logical
                              │
                              ▼
                    ┌────────────────────────┐
                    │   VDO dedup + compress    │
                    │   (inline, block layer)   │
                    └────────────────────────┘
                              │
                              ▼
        Physical disk actually stores:  ~1.1GB   (if data is highly redundant)
                                          ─ or ─
                                         ~4.9GB   (if data turns out NOT
                                                    reducible — e.g. already
                                                    compressed / encrypted)

  Logical size presented to the filesystem (50G) is FIXED regardless —
  only vdostats tells you which of the two physical-usage scenarios above
  you're actually living in.
```

### Implementation (Learn by Applying)

**Scenario:** You're setting up a backup target volume expected to hold highly redundant data (daily full backups with lots of unchanged content between runs).

```bash
dnf install -y vdo kmod-kvdo

vdo create --name=vdo_backup --device=/dev/sdi --vdoLogicalSize=50G

mkfs.xfs -K /dev/mapper/vdo_backup
mkdir -p /backups
mount /dev/mapper/vdo_backup /backups

vdostats --human-readable
```

```bash
for i in $(seq 1 5); do cp /var/log/messages /backups/backup_copy_$i.log; done
vdostats --human-readable
```

### Interview Questions — with Answers

**1. What's the practical difference between VDO's logical-vs-physical size gap and LVM thin provisioning's over-commit — are they the same risk in different clothing, or genuinely different?**

They're related but driven by different mechanisms. LVM thin provisioning's over-commit risk is purely about *allocation* — multiple volumes claiming more logical space than physically exists, with the risk materializing based on how much data actually gets written regardless of its content. VDO's risk is about *data reducibility* — the logical-to-physical gap exists because VDO is betting that incoming data will deduplicate/compress well; if the actual data turns out to be mostly unique or already-compressed (and thus doesn't reduce), you can run out of physical space even with relatively little logical data written, which is a distinct failure trigger from simple over-allocation. Both ultimately produce the same symptom (physical space exhausted despite logical space appearing available), but the root cause and the thing you'd monitor to predict it differ — allocation totals for thin provisioning, actual observed savings ratio for VDO.

**2. What kind of workload is a good candidate for VDO, and what kind of workload would see little to no benefit — or even a performance penalty?**

Good candidates: backup repositories (especially incremental/full backup chains with lots of unchanged data between runs), VM image stores (many VMs sharing a common OS base image have huge duplicate content), and general-purpose file shares with redundant content. Poor candidates: already-compressed data (video, most image formats, already-gzipped archives) which gains nothing from further compression and just adds CPU overhead for no benefit, and genuinely high-entropy/unique data (encrypted data, random data, most database workloads with unique row content) where deduplication finds little to nothing to save and you're left paying VDO's inline processing overhead purely as a cost with no offsetting capacity benefit.

**3. If `vdostats` shows physical space is running low despite the logical volume showing plenty of "free" space, what does that tell you, and what are your options?**

It tells you the actual data being written isn't reducing (deduplicating/compressing) as well as was assumed when the logical size was set — you're approaching genuine physical exhaustion even though the filesystem/application layer still believes there's plenty of room, because they only see the logical size. Options: add more physical backing storage to the VDO volume if the underlying device supports growth, identify and remove/archive data that isn't actually benefiting from VDO (defeating the purpose of storing it there), or in the worst case, migrate the workload to non-VDO storage if it turns out this workload was simply never a good VDO candidate to begin with.

**4. Why might compression/deduplication at the VDO (block) layer behave differently in practice than compression already happening at the application layer (e.g., a database that compresses its own data)?**

If the application layer has already compressed the data before it's written, the data arriving at VDO is high-entropy (compressed data looks essentially random at the byte level) and has very little further redundancy for VDO to find — you end up paying VDO's processing overhead for essentially zero additional savings, since you can't meaningfully compress already-compressed data. This is a real interview trap: stacking VDO under a workload that already compresses at the application layer is a common mistake that looks reasonable on paper ("more compression must be better") but delivers no benefit while still costing CPU cycles.

**5. Would you deploy VDO underneath LVM, or LVM underneath VDO, or avoid stacking them entirely — and why?**

The standard supported pattern is VDO underneath LVM — VDO presents its logical device, and LVM treats that VDO device as a PV, letting you keep all of LVM's normal flexibility (snapshots, resizing, pooling) on top of VDO's space-saving benefit underneath. Putting LVM underneath VDO (VDO consuming an LVM LV as its backing device) is less common but technically workable in some setups depending on the exact use case; the VDO-under-LVM ordering is generally the better-supported, more standard choice for typical use. I would avoid VDO entirely (not stack it at all) for workloads I've already identified as poor dedup/compression candidates, rather than stacking it "just in case" — the processing overhead isn't free, and there's no benefit to paying it without a reasonable expectation of savings.

---

## Topic 9: DM-Multipath — Redundant Paths to SAN Storage

### Quick Review
- DM-Multipath presents multiple physical paths to the same SAN LUN as one logical device (`/dev/mapper/mpathX`).
- Before multipathd claims a LUN, each path shows as a **separate** raw device in `lsblk`.
- `multipath -ll` is the primary diagnostic command — read path group status (`active`/`ready`/`faulty`/`ghost`).
- Path-level failures often surface in kernel/FC logs (`journalctl -k`, `dmesg`) before `multipath -ll` reports a path as faulty.

### Quick Learning

You've likely *used* multipathed SAN storage extensively given your background — the interview bar here isn't "have you seen this before," it's "can you diagnose it when it's degraded but not fully down." A LUN with one dead path out of two can still serve I/O perfectly fine through the surviving path, meaning applications notice nothing wrong — which is exactly why proactively checking `multipath -ll` and correlating with kernel-level FC events, rather than waiting for an application to complain, is the actual skill being tested.

**One LUN, two physical paths, one logical device:**
```
        RHEL Server
     ┌───────┴───────┐
   HBA1              HBA2
     │                 │
     ▼                 ▼
  Fabric A          Fabric B        <- independent switches/fabrics
     │                 │               for real redundancy
     └───────┬───────┘
              ▼
        SAN Array  ──▶  LUN0

  What the kernel sees BEFORE multipathd:
     /dev/sdc   (LUN0 via HBA1/Fabric A)
     /dev/sdd   (LUN0 via HBA2/Fabric B)     <- same LUN, two devices

  What multipathd presents AFTER claiming them (same WWID):
     /dev/mapper/mpath0
        ├─ path: sdc   status: active
        └─ path: sdd   status: ready (standby, active-passive example)

  One physical path (say, HBA1) fails ⇒ sdc drops to "faulty," but
  mpath0 keeps serving I/O through sdd — application sees NOTHING.
  This is exactly why you check `multipath -ll` proactively.
```

### Implementation (Learn by Applying)

**Scenario:** You're handed a RHEL server connected to SAN storage with two HBA paths per LUN. Verify multipath is actually configured correctly, and practice diagnosing a degraded path.

```bash
dnf install -y device-mapper-multipath
systemctl enable --now multipathd

lsblk
multipath -ll
```

```bash
multipath -v3
cat /etc/multipath.conf
multipath -t

systemctl reload multipathd
multipath -r

multipath -ll | grep -i fault
```

```bash
journalctl -k | grep -i multipath
journalctl -u multipathd
dmesg | grep -iE "rport|fc_remote|scsi.*offline"
```

### Interview Questions — with Answers

**1. Before multipathd claims a LUN's paths, what does `lsblk` show — one device or multiple? Why, and what does that tell you about how multipath actually works at the block layer?**

`lsblk` shows *multiple* separate block devices — one per physical path to the LUN (e.g., `/dev/sdc` and `/dev/sdd` might both be the same underlying LUN, seen via two different HBA/fabric paths). This tells you multipath isn't magic at the SCSI layer — the kernel genuinely sees the same LUN presented multiple times, once per path, and it's `multipathd`'s job, using the LUN's WWID to recognize these are the same underlying storage, that unifies them into a single logical `/dev/mapper/mpathX` device for applications to actually use. Understanding this raw-multiple-devices reality is exactly what lets you troubleshoot at the SCSI level when multipath itself seems confused about a LUN.

**2. Walk me through reading `multipath -ll` output for a LUN you've never seen before — what tells you it's healthy versus degraded, in under 30 seconds?**

I'd look at the path group structure first — how many groups, and which is marked active — then scan every individual path line within the active group for its status: `active` or `ready` paths are healthy, any path marked `faulty` or unexpectedly `ghost` (typically meaning that path is reachable but not the preferred/active one in an active-passive setup, which is often normal, not necessarily bad) needs a closer look. A fully healthy LUN with two paths should show both as `active`/`ready` in an active-active config, or one `active` and one `ready`/`ghost` (standby) in active-passive — a LUN with only one path listed at all, or any path explicitly marked `faulty`, is the signal something's degraded.

**3. A LUN is still accessible and the application hasn't noticed anything wrong, but you suspect one of two paths has silently failed. How do you confirm that, and where would you look beyond `multipath -ll` itself?**

`multipath -ll` is the first stop — a faulty path shows explicitly there. If it's ambiguous or `multipath -ll` hasn't caught up yet, I'd check kernel-level Fibre Channel/SCSI events directly: `journalctl -k` or `dmesg` filtered for remote-port (`rport`) state changes, SCSI device offline events, or HBA driver errors, since these often surface at the kernel/FC layer moments before multipathd's own health checks catch up and update `multipath -ll`'s reported state. I'd also check `journalctl -u multipathd` for the daemon's own logged path-checker activity, which can show a path failing its periodic health check even if the overall LUN status hasn't visibly changed yet from the application's perspective.

**4. What's the difference between an active-active and active-passive multipath configuration, and how would you tell which one you're looking at from the CLI output?**

In active-active, I/O is actively distributed across multiple paths simultaneously (often for load balancing/throughput), and `multipath -ll` typically shows multiple paths within a single "active" path group, all marked `active`. In active-passive, one path (or path group) actively handles all I/O while the other sits in standby, ready to take over on failure but not currently carrying traffic — `multipath -ll` shows this as two separate path groups, one marked `active` and the other `enabled` (standby), with paths in the standby group often showing as `ghost` rather than `active`. The array vendor and the LUN's configured path selection policy (visible in `multipath -ll`'s policy line, e.g., `round-robin` suggesting active-active load distribution) are the concrete clues.

**5. The SAN team says they've presented a new LUN, but it's not showing up under `multipath -ll`. Walk me through your troubleshooting steps, starting from the SCSI/HBA layer up through multipathd.**

First, confirm the HBA itself sees the new LUN at the SCSI layer — rescan the SCSI bus (`echo "- - -" > /sys/class/scsi_host/hostX/scan` for each relevant host, or the vendor-provided rescan script) and check `lsblk`/`lsscsi` for a new raw device appearing; if nothing shows up here, the problem is upstream of the OS entirely — zoning/masking on the SAN fabric side likely hasn't actually been completed despite what the SAN team reported. If the raw device DOES appear in `lsblk` but not in `multipath -ll`, the next check is whether `multipathd` has picked it up yet — `multipath -v3` gives a verbose rescan showing multipathd's actual decision process, and might reveal it's being deliberately excluded by a blacklist entry in `/etc/multipath.conf`. Finally, confirm the WWID multipathd is seeing matches what the SAN team expects for that LUN, since a mismatched or unexpected WWID (sometimes from an unusual array configuration) can cause multipathd to not group the paths as anticipated.

---

**End of Day 1.** You should now be able to build an LVM stack from raw disks, grow it live with zero downtime, explain and demonstrate exactly where and why shrinking breaks down for XFS, reason correctly about Copy-on-Write snapshot sizing and invalidation, speak concretely about disk-failure resilience and thin-pool over-commit risk, stand up and compare Stratis against classic LVM, apply VDO for deduplication-heavy workloads and understand its logical-vs-physical risk, and — critically for your background — actually diagnose (not just use) DM-Multipath SAN connectivity. You should also be able to defend every interview answer above out loud, in your own words, not just recognize it as correct when reading it.

Proceed to **Day 2 — Networking** next.
