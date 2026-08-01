 # Complete LVM Flow in RHEL 9

Excellent topic for RHCSA. Many people memorize commands like `pvcreate`, `vgcreate`, `lvcreate`, but don't actually understand **what each layer does**.

I'll explain it as if you were building a house.

---

# Complete LVM Flow

```text
Physical Disk
     │
     ▼
Partition (Optional but recommended)
     │
     ▼
Physical Volume (PV)
     │
     ▼
Volume Group (VG)
     │
     ▼
Logical Volume (LV)
     │
     ▼
Filesystem (XFS/EXT4)
     │
     ▼
Mount Point
     │
     ▼
Users/Application can use storage
```

Let's understand every layer.

---

# Step 1: Physical Disk

Imagine Linux detects a new disk.

```text
/dev/sdb
```

This is just a raw disk.

Think of it like buying a new empty hard drive.

```text
+---------------------+
|      /dev/sdb       |
|                     |
|    Empty Disk       |
+---------------------+
```

Check disks

```bash
lsblk
```

Example

```text
sda
├── sda1
└── sda2

sdb
```

Notice **sdb** has nothing.

---

# Step 2: Partition

Although LVM can use the entire disk directly, in enterprises we usually create a partition.

Example

```text
/dev/sdb1
```

Create partition

```bash
fdisk /dev/sdb
```

or

```bash
parted
```

After partitioning

```text
/dev/sdb
┌────────────────────────────┐
│          sdb1              │
└────────────────────────────┘
```

Tell Linux to reread the partition table

```bash
partprobe
```

Verify

```bash
lsblk
```

Output

```text
sdb
└── sdb1
```

---

# Why partition?

Because one disk can contain multiple partitions.

Example

```text
Disk = 500GB

sdb1 = 100GB
sdb2 = 200GB
sdb3 = 200GB
```

Each partition can become its own PV.

---

# Step 3: Physical Volume (PV)

Now convert the partition into an LVM disk.

```bash
pvcreate /dev/sdb1
```

What happens?

Linux adds LVM metadata.

```text
Before

sdb1

After

PV
```

Diagram

```text
+--------------------+
|      PV            |
| Metadata           |
| Storage Space      |
+--------------------+
```

Verify

```bash
pvs
```

or

```bash
pvdisplay
```

Example

```text
PV         VG
/dev/sdb1
```

Notice no VG yet.

---

# What is a PV?

Think of it as:

> "This storage is now managed by LVM."

Nothing more.

---

# Step 4: Volume Group (VG)

Now combine one or more PVs.

```bash
vgcreate vgdata /dev/sdb1
```

Diagram

```text
PV1
 │
 ▼

+------------+
|   vgdata   |
+------------+
```

Suppose later another disk arrives.

```text
/dev/sdc1
```

Create another PV

```bash
pvcreate /dev/sdc1
```

Extend VG

```bash
vgextend vgdata /dev/sdc1
```

Now

```text
PV1 ----\
          \
           VG
          /
PV2 -----/
```

Storage is pooled together.

Think of VG like a **storage pool**.

Example

Disk1 = 100GB

Disk2 = 200GB

```text
VG = 300GB
```

Applications never see individual disks.

They only see the pool.

Verify

```bash
vgs
```

or

```bash
vgdisplay
```

---

# Step 5: Logical Volume (LV)

Now create virtual partitions.

Example

```bash
lvcreate -L 50G -n lvdata vgdata
```

Diagram

```text
VG = 300GB

+---------------------------+
|                           |
| LV1 = 50GB                |
| LV2 = 100GB               |
| Free =150GB               |
|                           |
+---------------------------+
```

Linux creates

```text
/dev/vgdata/lvdata
```

or

```text
/dev/mapper/vgdata-lvdata
```

Verify

```bash
lvs
```

or

```bash
lvdisplay
```

---

# Why Logical Volume?

Unlike normal partitions

```text
/dev/sda1
```

an LV can

- Increase
- Decrease (with care, filesystem-dependent)
- Move
- Snapshot
- Span multiple disks

This is why enterprises use LVM.

---

# Step 6: Filesystem

Currently

```text
LV
```

is just empty blocks.

Need a filesystem.

Example

```bash
mkfs.xfs /dev/vgdata/lvdata
```

or

```bash
mkfs.ext4 /dev/vgdata/lvdata
```

Diagram

```text
LV

↓

Filesystem

↓

Files
Folders
Permissions
```

Without filesystem

```bash
touch file
```

won't work.

Verify

```bash
blkid
```

Output

```text
TYPE="xfs"
```

---

# Step 7: Mount Point

Create a directory

```bash
mkdir /data
```

Mount

```bash
mount /dev/vgdata/lvdata /data
```

Now

```text
Application

↓

/data

↓

Filesystem

↓

LV

↓

VG

↓

PV

↓

Disk
```

Verify

```bash
df -h
```

---

# Step 8: Permanent Mount

Edit

```text
/etc/fstab
```

Example

```fstab
/dev/vgdata/lvdata   /data   xfs   defaults   0 0
```

Test

```bash
mount -a
```

No output = success.

---

# Complete Picture

```text
              Physical Disk

                  /dev/sdb

                     │
                     ▼

             Partition (/dev/sdb1)

                     │
                     ▼

          pvcreate /dev/sdb1

                     │
                     ▼

         Physical Volume (PV)

                     │
                     ▼

vgcreate vgdata /dev/sdb1

                     │
                     ▼

      Volume Group (Storage Pool)

                     │
                     ▼

lvcreate -L 50G -n lvdata vgdata

                     │
                     ▼

      Logical Volume (Virtual Disk)

                     │
                     ▼

mkfs.xfs /dev/vgdata/lvdata

                     │
                     ▼

        Filesystem (XFS)

                     │
                     ▼

mkdir /data

mount /dev/vgdata/lvdata /data

                     │
                     ▼

      Users and Applications
```

---

# What Happens During Unmount?

Suppose `/data` is mounted.

```text
/data

↓

Filesystem

↓

LV
```

Run

```bash
umount /data
```

Now

```text
Application

X

Filesystem is detached
```

**Important:** `umount` does **not** delete data. It simply disconnects the filesystem from the directory tree. The data remains safely stored on the LV.

---

# Life Cycle of an LVM Volume

```text
Disk Added
     │
     ▼
Partition
     │
     ▼
PV Created
     │
     ▼
VG Created
     │
     ▼
LV Created
     │
     ▼
Filesystem Created
     │
     ▼
Mounted
     │
     ▼
Used by Applications
     │
     ▼
Unmounted (if needed)
     │
     ▼
Filesystem Removed (optional)
     │
     ▼
LV Removed (optional)
     │
     ▼
VG Removed (optional)
     │
     ▼
PV Removed (optional)
```

---

# RHCSA (EX200) Command Order to Remember

```bash
# 1. Create partition
fdisk /dev/sdb
partprobe

# 2. Create Physical Volume
pvcreate /dev/sdb1

# 3. Create Volume Group
vgcreate vgdata /dev/sdb1

# 4. Create Logical Volume
lvcreate -L 500M -n lvdata vgdata

# 5. Create Filesystem
mkfs.xfs /dev/vgdata/lvdata

# 6. Create Mount Point
mkdir /mnt/data

# 7. Mount
mount /dev/vgdata/lvdata /mnt/data

# 8. Make Persistent
echo "/dev/vgdata/lvdata /mnt/data xfs defaults 0 0" >> /etc/fstab

# 9. Test
mount -a
```

---

| Layer                    | One can contain many?    | Example                  |
| ------------------------ | ------------------------ | ------------------------ |
| Disk → Partitions        | ✅ Yes                    | `/dev/sdb1`, `/dev/sdb2` |
| Partition → PV           | ❌ No                     | One partition = One PV   |
| PV → VG                  | ❌ No                     | One PV belongs to one VG |
| VG → PV                  | ✅ Yes                    | VG can include many PVs  |
| VG → LV                  | ✅ Yes                    | Many LVs inside one VG   |
| LV → Filesystem          | Usually one              | One filesystem per LV    |
| Filesystem → Mount Point | Usually one active mount | Mounted at `/data`       |


# RHCSA Practice Question

A new disk `/dev/sdc` has been added to your RHEL 9 system.

Create a **1 GiB** logical volume named **lvproject** in a volume group named **vgproject** using a partition on the disk.

- Format it with XFS.
- Mount it permanently at `/project`.
- Verify that it survives a reboot.

Write the commands in order without looking at your notes.
