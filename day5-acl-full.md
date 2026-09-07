# Day 5 — Permissions Deep Dive: Access Control Lists (Full Study Edition, with Answers)

**Estimated time: 4–5 hours**

Day 4 covered the standard Unix permission model to its full depth — umask, SUID, SGID, sticky bit. Today is about the moment that model genuinely runs out of expressive power, and what you reach for instead: POSIX ACLs. This is a topic product-company interviewers use specifically to separate "read the RHCSA objectives once" from "actually had to solve a real access-control problem that plain `chmod` couldn't express" — the questions below are calibrated for that gap.

---

## Topic 1: ACL Fundamentals — When Permissions Alone Aren't Enough

### Quick Review
- Standard Unix permissions express exactly **one owner + one group + everyone else** — nothing more granular.
- ACLs let you grant specific permissions to **additional, named** users or groups beyond the standard three.
- `setfacl -m` modifies, `getfacl` reads; a `+` after the mode bits in `ls -l` is your visual cue an ACL exists.
- ACLs are stored as **extended attributes** on the filesystem (covered in depth in Topic 5) — the filesystem must support them (XFS and ext4 both do, by default on RHEL).

### Quick Learning

The standard Unix permission model has a hard structural limit: exactly three permission sets — owner, group, other — and nothing in between. The classic scenario that breaks this model: a shared file needs User A to have read-write, User B to have read-only, and neither of them shares a convenient group relationship you're willing to restructure just for this one file. Standard `chmod` genuinely cannot express this — you'd need a THIRD entity type between "group" and "other" that doesn't exist in the base model. ACLs exist to solve exactly this gap, attaching an arbitrary number of additional user/group-specific permission entries to a single file or directory.

**Where the standard model's expressiveness actually ends:**
```
  Standard Unix permissions — exactly 3 buckets, no more:
     Owner:  rwx
     Group:  rwx     (ONE group only)
     Other:  rwx     (everyone else, no distinctions possible)

  Scenario: file needs...
     - alice:  rw-        (alice is not in any convenient shared group)
     - bob:    r--         (bob needs LESS access than alice)
     - everyone else: ---  (no access at all)

  chmod CANNOT express this — "group" can only be ONE group, and
  there's no way to give alice and bob DIFFERENT levels of access
  without restructuring group membership entirely (and even then,
  a single group still can't give alice and bob DIFFERENT permissions
  from each other — they'd share whatever the group grants)

  ACLs add exactly the missing capability:
     setfacl -m u:alice:rw- file
     setfacl -m u:bob:r--   file
        │
        ▼
  getfacl file
     # file: file
     # owner: root
     # group: root
     user::rw-
     user:alice:rw-        <- named user, INDEPENDENT of alice's own group
     user:bob:r--           <- named user, independently scoped from alice
     group::---
     mask::rw-
     other::---
```

### Implementation (Learn by Applying)

**Scenario:** A finance report file needs `alice` to have read-write access and `bob` to have read-only access, with everyone else having no access at all — and neither user shares a convenient group. Solve it with ACLs, then observe the visual indicator ACLs leave on standard `ls -l` output.

```bash
touch /finance_report.xlsx
chmod 600 /finance_report.xlsx          # Start from "only the owner has access, nothing else"
chown root:root /finance_report.xlsx

setfacl -m u:alice:rw- /finance_report.xlsx
setfacl -m u:bob:r-- /finance_report.xlsx

getfacl /finance_report.xlsx
ls -l /finance_report.xlsx              # Note the "+" after the mode bits — that's your ACL indicator
```

Verify it actually works as intended:
```bash
su - alice -c "echo 'test edit' >> /finance_report.xlsx"    # Should succeed
su - bob -c "cat /finance_report.xlsx"                       # Should succeed (read-only)
su - bob -c "echo 'unauthorized' >> /finance_report.xlsx"    # Should FAIL — bob only has read
su - charlie -c "cat /finance_report.xlsx"                   # Should FAIL — charlie has no ACL entry at all, falls to "other"
```

Remove a specific entry or all ACLs cleanly:
```bash
setfacl -x u:bob /finance_report.xlsx   # Remove just bob's entry
getfacl /finance_report.xlsx

setfacl -b /finance_report.xlsx         # Strip ALL ACL entries, back to plain Unix permissions
ls -l /finance_report.xlsx              # The "+" is gone
```

### Interview Questions — with Answers

**1. Give a concrete, real-world scenario where standard Unix permissions genuinely cannot express what's needed, and explain specifically why — not just "ACLs are more flexible."**

A shared file or directory that needs three or more DIFFERENT levels of access for different individual users who don't share a clean, dedicated group relationship — for example, a contractor needing read-only access to a project file, a project lead needing full read-write, and a auditor needing read-only access but for compliance reasons should NOT be lumped into the same group as the contractor (since group membership might grant unrelated access elsewhere). Standard permissions offer exactly one "group" slot — you cannot express "these two specific people get read access, but this other specific person gets read-write, and they're not otherwise relatable via a single convenient group" without either creating an excessive number of single-purpose groups (which becomes its own management burden) or using ACLs, which let you attach these individual, named permission grants directly to the file itself.

**2. What does the `+` at the end of a file's permission string in `ls -l` output actually indicate, and why is it easy to overlook during a routine permissions audit?**

The `+` indicates the file has ACL entries beyond the standard owner/group/other permissions — it's the ONLY visual signal `ls -l` gives you that there's more to this file's actual effective permissions than what the basic `rwxrwxrwx`-style string shows. It's easy to overlook during an audit because someone scanning permissions quickly, especially via scripts or tools that only parse the standard mode bits, can completely miss that a file has ADDITIONAL access granted via ACL — a file that LOOKS appropriately locked down based on its base permissions alone (say, `rw-------`) could actually have a very permissive ACL entry granting broad access to another user or group, invisible unless you specifically run `getfacl` or notice that easily-missed `+`.

**3. Walk me through your debugging steps: a user reports they can't access a file that `ls -l` shows they SHOULD have access to based on the group ownership shown.**

First, I'd check for the `+` indicator on `ls -l` output — if present, the base permission string alone doesn't tell the full story, and I'd run `getfacl` on the file to see the complete, actual effective permission set including any ACL entries that might be overriding what the base group permission would otherwise suggest. I'd specifically check for a named ACL entry for that exact user (which, depending on configuration, can sometimes restrict access even below what the base group permission implies, if explicitly set that way) and check the ACL mask entry (covered in depth in Topic 3), since the mask can silently cap the EFFECTIVE permission of every named ACL entry below what's individually listed, producing exactly this kind of "should have access but doesn't" confusion.

**4. What's the difference between `setfacl -m` and `setfacl -x`, and what does `setfacl -b` do differently from both?**

`setfacl -m` (modify) adds or updates a specific ACL entry — you specify the user/group and the permission, and it's added to or updated within the existing ACL. `setfacl -x` (remove) deletes one SPECIFIC named entry from the ACL, without touching any other existing entries. `setfacl -b` is a full reset — it removes ALL ACL entries from the file entirely, restoring it to relying purely on standard Unix owner/group/other permissions with no ACL entries left at all, which is a much broader operation than removing a single entry with `-x`.

**5. If ACLs can express essentially the same thing that creating many fine-grained groups could theoretically achieve, why do teams generally prefer ACLs for this kind of one-off, per-file access requirement rather than just creating more groups?**

Creating a dedicated group for every unique combination of "these specific people need this specific access to this specific file" doesn't scale — you'd end up with an ever-growing number of single-purpose, file-specific groups that are hard to track, hard to remember the purpose of months later, and create real management overhead (group creation, membership changes, eventual cleanup when no longer needed). ACLs let you express the exact same access requirement directly on the file itself, visible via `getfacl` right there on the object it applies to, without needing to invent and maintain a whole separate group just to serve one specific, often temporary, access need — for genuinely broad, long-term, org-wide access patterns, dedicated groups still make sense, but for narrow or one-off requirements, ACLs are the more maintainable, more discoverable solution.

---

## Topic 2: Default ACLs & Inheritance

### Quick Review
- A **default ACL** (`setfacl -m d:...`) applies only to **directories**, and is automatically inherited by every NEW file/subdirectory created inside.
- Default ACLs do NOT retroactively apply to files that already existed before the default ACL was set.
- Default ACL inheritance happens **regardless of the creating user's umask** — this is exactly the more robust alternative referenced at the end of Day 4's UMASK topic.
- A directory's default ACL and its own regular (access) ACL are two SEPARATE things — you need `getfacl` and read the output carefully to distinguish `default:` entries from regular ones.

### Quick Learning

Regular ACL entries (Topic 1) only affect the exact file/directory you applied them to. Default ACLs solve a different, forward-looking problem: ensuring every FUTURE file created in a directory automatically gets the right permission structure, without depending on every team member's personal umask being configured correctly — directly closing the gap flagged at the end of yesterday's UMASK topic. This is the mechanism that makes shared team directories genuinely maintainable at scale, rather than requiring someone to manually re-apply ACLs to every new file as it's created.

**Regular ACL vs. default ACL — two entirely separate things on the same directory:**
```
  getfacl /shared/teamdir
  ────────────────────────
  # file: shared/teamdir
  # owner: root
  # group: devteam
  user::rwx
  group::rwx
  other::---
  user:contractor:r-x                    <- REGULAR ACL entry: applies to
                                              THIS directory itself only
  default:user::rwx
  default:group::rwx
  default:group:auditors:r-x             <- DEFAULT ACL entry: gets
  default:other::---                        automatically COPIED onto
                                              every NEW file/subdir
                                              created inside, going forward

  alice creates /shared/teamdir/newfile.txt
        │
        ▼
  newfile.txt automatically INHERITS the default ACL entries
  as its OWN regular ACL — including default:group:auditors:r-x
  becoming a real, active group:auditors:r-x entry ON newfile.txt
  itself, with ZERO action required from alice, regardless of
  alice's personal umask setting
```

### Implementation (Learn by Applying)

**Scenario:** Build a shared compliance directory where an auditor group must automatically have read access to every file anyone creates, forever, without depending on individual umask discipline — and demonstrate that default ACLs are NOT retroactive for pre-existing files.

```bash
groupadd auditors
usermod -aG auditors carol

mkdir /compliance
chown root:devteam /compliance
chmod 2770 /compliance                  # SGID for group inheritance (Day 4), base access for devteam only

# Set a DEFAULT ACL so auditors automatically get read access to everything created here going forward
setfacl -m d:group:auditors:r-x /compliance
getfacl /compliance                     # Note the "default:" prefix on this entry specifically
```

Prove inheritance works for NEW files, and prove it does NOT retroactively apply to a file that already existed:
```bash
su - root -c "touch /compliance/preexisting_file.txt"    # Created BEFORE the default ACL... wait, reorder for clarity below
```
Reorder to demonstrate correctly:
```bash
touch /compliance/before_default_acl.txt      # (Imagine this existed prior to setfacl -m d:... above)
touch /compliance/after_default_acl.txt        # Created AFTER the default ACL is in place

getfacl /compliance/before_default_acl.txt     # No auditors entry — default ACLs are NOT retroactive
getfacl /compliance/after_default_acl.txt      # HAS the inherited auditors:r-x entry automatically

su - carol -c "cat /compliance/after_default_acl.txt"     # Succeeds — inherited ACL grants it
su - carol -c "cat /compliance/before_default_acl.txt"    # Fails — this file predates the default ACL
```

Fix the pre-existing file by applying the same entry retroactively where needed:
```bash
setfacl -m g:auditors:r-x /compliance/before_default_acl.txt
```

### Interview Questions — with Answers

**1. Explain why default ACLs are described as "not retroactive" — walk through exactly what that means with a concrete before/after example.**

A default ACL only defines what gets COPIED onto new files/subdirectories at the MOMENT they're created inside that directory — it has no mechanism for reaching back and modifying files that already existed in the directory before the default ACL was set, because those files' permissions were already finalized at their own creation time, under whatever rules were in effect THEN. Concrete example: if `/compliance/old_report.txt` was created last month, and today you set a default ACL granting the `auditors` group read access on the `/compliance` directory, `old_report.txt` will NOT automatically gain that auditors access — only files created AFTER today's `setfacl -m d:...` command will inherit it; `old_report.txt` needs its own explicit `setfacl -m g:auditors:r-x` applied directly if you want it to have the same access retroactively.

**2. How does default ACL inheritance interact with a creating user's personal umask setting — does a restrictive umask override or interfere with default ACL inheritance?**

Default ACL inheritance operates independently of, and takes precedence in a specific sense over, the creating user's umask for the ACL-defined permissions specifically — the whole point of default ACLs is to guarantee a specific, consistent permission structure regardless of what any individual user's umask happens to be configured as, which is exactly why they're the more robust solution referenced when a shared directory can't rely on every team member's personal umask being correctly and consistently set. That said, the umask still interacts with the STANDARD (non-ACL) portion of the new file's permissions in the usual way — the ACL-inherited entries specifically bypass that dependency for the explicitly ACL-covered users/groups, which is the actual value proposition of using default ACLs over relying on umask alone.

**3. What's the difference between a "regular" (access) ACL entry and a "default" ACL entry on the SAME directory, and why can a directory have both simultaneously with different content?**

A regular (access) ACL entry governs actual access TO that directory itself — who can list its contents, traverse into it, write to it directly, exactly like a regular ACL entry on a file. A default ACL entry doesn't grant any access to the directory itself at all — it's purely a TEMPLATE that gets copied onto new files/subdirectories created within it. A directory commonly has both simultaneously with genuinely different content because the access needs for the directory itself (who can browse/write here) and the inheritance policy for what gets created inside it (what should every new file automatically look like) are related but distinct concerns — for example, a directory's regular ACL might grant a specific contractor read-only access to browse it, while its default ACL ensures every NEW file created by team members automatically grants read access to an auditor group, which is a completely separate requirement from the contractor's own browsing access.

**4. You set a default ACL on a directory, but new subdirectories created inside it don't seem to be passing the inheritance further down to files created inside THEM. What's the likely issue?**

Default ACLs are inherited by new files AND new subdirectories as their own REGULAR (access) ACL entries — but for a new subdirectory to continue propagating that same inheritance behavior further down to things created inside IT, the default ACL entries also need to be copied onto the new subdirectory as ITS OWN default ACL entries too, not just as its regular access ACL. In practice, `setfacl`'s default ACL inheritance does correctly copy default entries onto new subdirectories as both their access AND their own default ACL simultaneously by design — so if this genuinely isn't propagating, I'd check whether someone later ran `setfacl -k` (which specifically removes default ACL entries) on an intermediate subdirectory, breaking the inheritance chain at that specific point, or whether the subdirectory was actually created via a method that bypasses normal filesystem operations (like being moved/rsynced in from elsewhere with `--no-acls`, rather than genuinely created fresh inside the parent directory).

**5. How would you audit an entire directory tree to find every file/directory that has ACL entries at all, distinguishing them from files relying purely on standard permissions?**

`getfacl -R -s /path/to/directory` recursively walks the tree and, with `-s` (skip-base), specifically SKIPS files that only have the standard base entries with no actual additional ACL — meaning the output only shows files/directories that genuinely have meaningful, non-default ACL content, making it far easier to scan a large tree for actual ACL usage rather than getting flooded with the base-only entries every file technically has. Alternatively, a targeted `find` approach: `find /path -type f -exec getfacl -c {} \; 2>/dev/null | grep -B5 "^user:\|^group:"` or checking for the `+` indicator directly via a script parsing `ls -l` output across the tree — but `getfacl -R -s` is the cleaner, purpose-built approach for exactly this audit task.

---

## Topic 3: The ACL Mask Entry — The Most Misunderstood Part of ACLs

### Quick Review
- The **mask** is a CEILING on the effective permissions of every named user/group ACL entry (and the owning group entry) — NOT a permission grant itself.
- `setfacl -m` automatically recalculates the mask by default when you add a new named entry, unless you explicitly manage it yourself.
- A named ACL entry showing `rwx` in `getfacl` output can still be EFFECTIVELY restricted if the mask is lower — always check the `effective:` annotation.
- `chmod g=` on a file WITH an ACL actually modifies the MASK, not the base group permission — a frequently-misunderstood interaction.

### Quick Learning

This is the single ACL concept most candidates get wrong, and it's exactly why interviewers ask about it specifically: the mask doesn't grant anything — it CAPS what every named ACL entry is actually allowed to use, regardless of what that entry itself says. A named user entry showing `rwx` in `getfacl` can be completely neutered down to `r--` in actual effective permission if the mask is set to `r--`, and this happens SILENTLY — `getfacl` will show the discrepancy via an `#effective:` annotation, but if you're not looking for it, you'll misread the ACL entirely.

**The mask as a ceiling, not a grant:**
```
  getfacl /project_file
  ─────────────────────
  user::rwx
  user:alice:rwx            <- alice's NAMED entry SAYS rwx...
  group::rwx
  mask::r-x                 <- ...but the MASK caps at r-x
  other::---

  Effective permission for alice = user:alice entry AND mask, combined:
     rwx  (alice's entry)
   & r-x  (the mask)
   ───────
     r-x  (ACTUAL effective permission — alice does NOT have write,
           despite her own ACL entry literally saying rwx)

  getfacl actually WARNS you about this directly:
     user:alice:rwx          #effective:r-x
                              ▲
                              this annotation only appears when the
                              mask is actually restricting something —
                              if you don't see "#effective:" on a line,
                              that entry's full listed permission IS
                              what's actually granted
```

### Implementation (Learn by Applying)

**Scenario:** Reproduce the exact "why doesn't my ACL grant work" confusion caused by the mask, understand why it happened, and fix it correctly — then observe how `chmod` on an ACL'd file interacts with the mask rather than the base group permission, a subtlety that trips up even experienced admins.

```bash
touch /project_file
setfacl -m u:alice:rwx /project_file
getfacl /project_file           # At this point, mask is likely rwx too (auto-calculated), alice genuinely has rwx

# Now someone runs a seemingly-unrelated chmod to "tighten group access"
chmod 750 /project_file          # This looks like it should leave alice's rwx untouched... does it?
getfacl /project_file            # Observe: mask has been recalculated, likely now r-x, CAPPING alice's entry
```
This is the crucial, frequently-misunderstood interaction: `chmod`'s group-permission field on a file that HAS an ACL doesn't set the base group permission the way it would on a plain file — it actually rewrites the ACL MASK, because the mask is what `ls -l`'s displayed "group" column represents once an ACL is present. Someone running a routine `chmod` expecting to only affect the base group entry can inadvertently cap every named ACL entry on the file without realizing it.

```bash
su - alice -c "echo test >> /project_file" 2>&1    # Now fails, despite alice's ACL entry still literally showing rwx

# Fix: explicitly recalculate/set the mask to what's actually intended
setfacl -m mask::rwx /project_file
getfacl /project_file
su - alice -c "echo test >> /project_file"          # Now succeeds again
```

### Interview Questions — with Answers

**1. Explain what the ACL mask actually does, in your own words, and why calling it a "permission grant" is technically incorrect.**

The mask defines the MAXIMUM effective permission that any named user entry, named group entry, or the owning group entry can actually exercise — it functions as a ceiling applied on TOP of whatever those individual entries themselves specify, not as a grant of access in its own right. Calling it a "grant" is incorrect because setting the mask to `rwx` doesn't GIVE anyone any access by itself — it simply doesn't restrict anything further; the actual access still has to come from an individual named entry (or the base owner/other entries, which the mask doesn't affect at all) — the mask can only take away from what's already specified elsewhere, never add to it.

**2. A named ACL entry for a user shows `rwx` in `getfacl` output, but that user reports they can't actually write to the file. What's the most likely explanation, and how would you confirm it from the `getfacl` output itself?**

The most likely explanation is that the ACL mask is set lower than the named entry — for example, a mask of `r-x` capping an entry that itself says `rwx`, meaning the user's EFFECTIVE permission is actually the intersection of their entry and the mask, not the entry alone. I'd confirm this directly from `getfacl` output by looking for an `#effective:` annotation appended to that specific line — `getfacl` explicitly flags any entry where the mask is actively restricting it below its own listed value, so the presence (or absence) of that annotation immediately confirms or rules out the mask as the cause without needing to manually cross-reference the mask value against every entry myself.

**3. Why does running a routine `chmod` on a file that already has an ACL sometimes unexpectedly break access for users who have explicit ACL entries?**

Once a file has an ACL, `chmod`'s group-permission field no longer directly modifies a simple base group permission the way it does on a plain file without an ACL — instead, it rewrites the ACL MASK, since the mask is what `ls -l`'s standard group-permission display column actually represents for an ACL'd file. Someone running `chmod 750` expecting only to adjust the base group permission, with no awareness the file even has an ACL (easy to miss without noticing the `+` in `ls -l`), can inadvertently recalculate the mask to something more restrictive than intended — silently capping every named ACL entry's effective permission below what those entries individually specify, breaking access for users who were explicitly granted it via ACL, without the `chmod` command itself giving any indication that's what just happened.

**4. When you run `setfacl -m u:newuser:rwx file` on a file that already has other ACL entries, does the mask get automatically recalculated? Why does this matter operationally?**

By default, yes — `setfacl` automatically recalculates the mask to the UNION of all entries' permissions whenever you modify the ACL, unless you explicitly manage the mask yourself (e.g., using `setfacl -n` to prevent automatic mask recalculation, or explicitly setting the mask value as part of the same operation). This matters operationally because relying on the automatic recalculation means the mask's actual restrictive effect can shift unpredictably every time ANYONE modifies the ACL on that file — a mask that was deliberately set restrictively for a security reason can get silently widened back out the next time someone adds an unrelated ACL entry, unless the team is specifically aware of and managing this auto-recalculation behavior rather than assuming the mask stays fixed once set.

**5. How would you deliberately set a restrictive mask and PREVENT it from being automatically widened the next time someone adds a new ACL entry to the same file?**

I'd use `setfacl -n` (no-mask-recalculate) when adding new entries going forward, which explicitly tells `setfacl` not to automatically recompute the mask based on the union of all entries — preserving whatever mask value is currently set even as new named entries are added. I'd also explicitly document/communicate this non-standard behavior to the team managing that file/directory, since it deviates from `setfacl`'s normal automatic-recalculation default, and anyone unaware of the `-n` convention in use could still inadvertently widen the mask by running a plain `setfacl -m` without the flag — this is a case where consistent team practice/documentation matters as much as the technical mechanism itself.

---

## Topic 4: Troubleshooting & Managing ACLs in Practice

### Quick Review
- Not all tools/operations preserve ACLs by default — `cp` needs `-p` (or `--preserve=all`), `rsync` needs `-A`, `tar` needs `--acls`.
- `mount` options matter: an ACL-supporting filesystem must actually be mounted with ACL support active (XFS defaults to ACL support on; older ext4 setups sometimes required explicit `acl` mount option, though modern RHEL defaults typically handle this correctly).
- Backup/restore and filesystem migration are the most common places ACLs get silently lost.
- `getfacl -R` and `setfacl --restore` let you snapshot and later restore an entire tree's ACL state, useful for migrations.

### Quick Learning

ACLs being silently dropped during a copy, backup, or migration is one of the most common real-world "why did our carefully-configured permissions disappear" incidents — and it's rarely because ACLs "broke," it's almost always because the specific tool/flags used for the operation simply weren't ACL-aware by default. This is exactly the kind of practical, operational knowledge that separates someone who's only ever run `setfacl` in a lab from someone who's actually had to migrate a real filesystem and account for everything living on it.

**Where ACLs commonly get silently lost:**
```
  cp file1 file2                    ✘  ACLs LOST (plain cp doesn't preserve them)
  cp -p file1 file2                 ✔  ACLs preserved (-p / --preserve=all)

  rsync -av src/ dst/               ✘  ACLs LOST (standard -av does NOT include ACLs)
  rsync -avA src/ dst/              ✔  ACLs preserved (-A specifically for ACLs)

  tar -cf backup.tar /data          ✘  ACLs LOST by default
  tar --acls -cf backup.tar /data   ✔  ACLs preserved

  Standard filesystem backup tools (varies by tool) — ALWAYS verify
  ACL-preservation behavior explicitly before trusting a backup/
  restore/migration workflow with ACL-dependent data, rather than
  assuming "it's just a file copy, permissions come along for free"
```

### Implementation (Learn by Applying)

**Scenario:** You're migrating the `/compliance` directory (with its ACLs from Topic 2) to new storage. Prove that a naive copy silently drops the ACLs, then perform the migration correctly and verify nothing was lost.

```bash
# Naive copy — demonstrate the problem first
mkdir /compliance_naive_copy
cp -r /compliance/* /compliance_naive_copy/
getfacl /compliance_naive_copy/after_default_acl.txt    # ACL entries are GONE — only base permissions survived
```

Correct migration approach:
```bash
# Snapshot the full ACL state of the source tree BEFORE migrating
getfacl -R /compliance > /root/compliance_acl_backup.txt

# Perform the actual copy/migration WITH explicit ACL preservation
rsync -avA /compliance/ /new_storage/compliance/
getfacl -R /new_storage/compliance | diff - <(getfacl -R /compliance)    # Should show no differences

# Alternative: if ACLs were somehow still lost, restore from the earlier snapshot
setfacl --restore=/root/compliance_acl_backup.txt
```

Verify the filesystem itself actually supports and has ACLs active:
```bash
mount | grep -w "/new_storage"          # Check mount options — look for acl explicitly, or confirm XFS (ACL-capable by default)
tune2fs -l /dev/vg_data/lv_newstorage 2>/dev/null | grep "Default mount options"   # ext4-specific check
```

### Interview Questions — with Answers

**1. A team migrates data to new storage using a standard `cp -r`, and afterward reports that some users have lost access they previously had. What's your first hypothesis, given what you know about ACLs?**

My first hypothesis is that the original files had ACL entries granting the affected users their access, and the plain `cp -r` operation — which does not preserve ACLs by default — silently dropped those entries during the copy, leaving only the base standard permissions behind. I'd confirm by checking `ls -l` for the `+` indicator on the ORIGINAL source files (if still available) or checking backup/documentation for known ACL usage, and comparing `getfacl` output between a known-good original (if it still exists somewhere) and the migrated copy — a mismatch showing ACL entries present on the original but absent on the migrated copy would confirm this exact root cause.

**2. What specific flags does `rsync` need to genuinely preserve ACLs during a copy, and why doesn't the commonly-used `-av` combination include this by default?**

`rsync` needs the explicit `-A` flag to preserve ACLs (often combined with `-X` for extended attributes more broadly, since ACLs are themselves implemented as extended attributes, covered in Topic 5). The common `-av` combination doesn't include ACL preservation by default because ACL support requires additional processing overhead and isn't universally needed for every rsync operation — the tool's designers made ACL preservation opt-in rather than assumed, meaning anyone running a standard `rsync -av` without specifically knowing to add `-A` will silently lose ACL data on the destination without any warning that this happened.

**3. Before trusting a full filesystem migration to preserve everything correctly, including ACLs, what verification steps would you take?**

I'd take a full `getfacl -R` snapshot of the source tree BEFORE migration as a definitive baseline, perform the migration using explicitly ACL-aware tooling/flags, and then take a second `getfacl -R` snapshot of the destination AFTER migration, diffing the two outputs directly to confirm byte-for-byte equivalence rather than just assuming the migration tool worked correctly because it didn't report any errors. I'd also spot-check actual access behavior for a sample of the specific users/groups known to have ACL-granted access, logging in as (or `su -`'ing to) those accounts and confirming they can genuinely still perform the operations their ACL entries were supposed to grant — verifying the ACL DATA survived is necessary but not sufficient; confirming the ACCESS BEHAVIOR still works as intended closes the loop completely.

**4. What's `setfacl --restore` for, and describe a real operational scenario where having previously run `getfacl -R > backup_file` saves you during an incident.**

`setfacl --restore` reads a file previously generated by `getfacl -R` (or similar) and reapplies every ACL entry it describes onto the corresponding files/directories, effectively restoring an entire tree's ACL state from a text-based snapshot. Real scenario: a well-intentioned but mistaken bulk `setfacl -b -R` (recursively STRIPPING all ACLs) gets run against the wrong directory tree by accident, instantly wiping out carefully configured, complex ACL structures across potentially hundreds of files — if a `getfacl -R` snapshot was taken beforehand (as a routine practice before any bulk ACL-affecting operation), `setfacl --restore` against that snapshot can fully recover the exact prior ACL state in one command, rather than needing to manually reconstruct every individual `setfacl -m` call that had originally been used to build up that structure over time.

**5. How would you confirm, on a brand new filesystem you've just created, that it actually supports and has ACL functionality active before relying on it in production?**

For XFS (the RHEL default), ACL support is built-in and active by default with no special mount option required — I'd simply test it directly: create a test file, run `setfacl -m u:someuser:rwx testfile`, then `getfacl testfile` to confirm the entry was actually accepted and is visible, rather than assuming based on filesystem type alone. For ext4, I'd check the mount options via `mount | grep <mountpoint>` for an explicit `acl` option, or check `tune2fs -l` for the filesystem's default mount options, since some ext4 configurations historically required this to be explicitly enabled (though modern RHEL defaults generally handle this correctly) — the direct functional test (actually setting and reading back an ACL entry) is the most reliable confirmation regardless of filesystem type, since it tests the real behavior rather than relying on assumptions about defaults that can vary by RHEL version or specific mount configuration.

---

## Topic 5 (Bonus — Advanced): Extended Attributes (xattrs) and NFSv4 ACLs

### Quick Review
- POSIX ACLs are themselves implemented as a specific category of **extended attributes (xattrs)** — `system.posix_acl_access` and `system.posix_acl_default`.
- `getfattr`/`setfattr` work with xattrs generally — ACLs are just one particular use of that broader mechanism.
- **NFSv4 ACLs** are a genuinely different, richer model than POSIX ACLs — not just POSIX ACLs "over NFS."
- When POSIX-ACL'd data is served over NFSv4, translation between the two models can lose fidelity — a real interoperability gotcha.

### Quick Learning

Understanding that ACLs are "just" a structured use of the more general extended-attribute mechanism explains several things at once: why `tar`/`rsync` need separate flags for ACLs specifically (they're xattr data, requiring xattr-aware handling, same as any other xattr like SELinux contexts), and why tools that generically handle "preserve all extended attributes" often cover ACLs as a side effect. Going one level further: NFSv4 introduced its OWN, richer ACL model — genuinely different from POSIX ACLs, not merely POSIX ACLs transported over a network protocol — with more granular permission types and different inheritance semantics, and understanding that distinction matters the moment POSIX-ACL'd data needs to be served over NFSv4 to non-Linux or mixed clients.

**ACLs as one specific application of the broader xattr mechanism:**
```
  Extended attributes (xattrs) — arbitrary name:value metadata
  attached to a file, beyond standard permissions/ownership:

     security.selinux         <- SELinux context (Day 7)
     system.posix_acl_access  <- POSIX ACL (what setfacl/getfacl manage)
     system.posix_acl_default <- POSIX default ACL
     user.custom_tag          <- arbitrary application-defined metadata

  getfattr -d -m - file        <- shows ALL extended attributes, not
                                   just the ACL-specific ones — this
                                   is the general-purpose tool; setfacl/
                                   getfacl are ACL-SPECIFIC front-ends
                                   over this same underlying mechanism
```

**POSIX ACLs vs. NFSv4 ACLs — genuinely different models, not the same thing over a wire:**
```
  POSIX ACL                              NFSv4 ACL
  ──────────                             ─────────
  A handful of permission bits           MANY distinct, granular
  (read/write/execute, roughly)          permission types (e.g.
                                          separate "delete" vs
                                          "delete_child", separate
                                          ACL-read vs ACL-write, etc.)

  Single "mask" entry caps ALL           No equivalent single-mask
  named entries at once                   concept — each entry stands
                                           on its own

  Default ACL inheritance (directory     ALLOW/DENY entries with
  → new children)                        explicit inheritance FLAGS
                                          per-entry (more granular
                                          control over WHAT inherits
                                          and how)

  When POSIX-ACL'd data is served over NFSv4, the server must
  TRANSLATE between these two genuinely different models — this
  translation is not always perfectly lossless, which is a real
  interoperability consideration for mixed environments (e.g.
  Linux POSIX-ACL'd storage served to Windows/other NFSv4 clients
  expecting the richer NFSv4 ACL semantics)
```

### Implementation (Learn by Applying)

**Scenario:** Directly inspect the extended attribute underlying a POSIX ACL to see the mechanism firsthand, then reason through the practical implications of serving POSIX-ACL'd data over NFSv4.

```bash
touch /xattr_demo_file
setfacl -m u:alice:rwx /xattr_demo_file

# See the ACL through the ACL-specific tool (what you've used all day)
getfacl /xattr_demo_file

# Now see the SAME information through the general-purpose extended attribute tool
getfattr -d -m - /xattr_demo_file
# Output includes something like: system.posix_acl_access=... (a binary-encoded blob
# representing exactly the same ACL data getfacl just displayed in human-readable form)

# Confirm the filesystem's xattr support generally (not just ACL-specific)
setfattr -n user.department -v "finance" /xattr_demo_file
getfattr -n user.department /xattr_demo_file
```

For the NFSv4 ACL comparison, if NFS server tooling is available in your lab:
```bash
dnf install -y nfs4-acl-tools

# NFSv4-native ACL tools (genuinely different command set/model from setfacl/getfacl)
nfs4_setfacl -a A::alice@localdomain:rwaDxtTcCy /xattr_demo_file
nfs4_getfacl /xattr_demo_file
```
Compare the NFSv4 ACL's far more granular permission letters (`r`ead, `w`rite, `a`ppend, `D`elete, `x`ecute, `t` (read attrs), `T` (write attrs), `c` (read ACL), `C` (write ACL), `y` (synchronize)) against POSIX ACL's simple `rwx` — this granularity difference is exactly what makes lossless translation between the two models non-trivial.

### Interview Questions — with Answers

**1. Explain what extended attributes (xattrs) are, in general, and where POSIX ACLs fit within that broader category.**

Extended attributes are arbitrary name-value metadata pairs a filesystem can attach to a file beyond its standard metadata (permissions, ownership, timestamps, size) — used for a variety of purposes including SELinux security contexts (`security.selinux`), POSIX ACLs (`system.posix_acl_access`/`system.posix_acl_default`), and arbitrary user-defined metadata (`user.*` namespace). POSIX ACLs are specifically ONE structured use case within this general mechanism — `setfacl`/`getfacl` are purpose-built, ACL-aware front-ends that read and write that specific `system.posix_acl_*` extended attribute in its defined binary format, while the more general `getfattr`/`setfattr` tools can read/write ANY extended attribute, ACL-related or otherwise, without necessarily understanding or formatting the ACL-specific binary encoding in the same human-readable way `getfacl` does.

**2. Why do backup/archive tools like `tar` and `rsync` need SEPARATE explicit flags for ACLs specifically, rather than ACLs just automatically coming along with a normal file copy?**

Because ACLs are stored as extended attributes, not as part of a file's standard, universally-copied metadata (like basic permission bits, which every copy operation handles by default) — a tool has to be SPECIFICALLY xattr-aware and specifically instructed to read and reproduce that extended attribute data on the destination, since xattrs in general (not just ACLs) are an optional, filesystem-dependent feature that not every copy operation assumes needs to be preserved by default. This is exactly why `tar --acls`, `rsync -A`, and `cp --preserve=all` (or the more general `-X`/xattr-preserving flags many tools offer) exist as explicit, opt-in options rather than ACL preservation being an automatic, unavoidable part of any file copy.

**3. What's genuinely different about NFSv4 ACLs compared to POSIX ACLs — is it really just "the same ACL concept, transported over NFS," or something more fundamentally different?**

It's genuinely a different, richer permission model, not merely POSIX ACLs carried over a network protocol — NFSv4 ACLs support many more distinct, granular permission types (separately distinguishing things like delete-this-object versus delete-a-child-within-this-directory, or separately controlling who can read versus write the ACL itself, none of which POSIX ACLs' simpler rwx-based model expresses), and use an explicit ALLOW/DENY entry model with per-entry inheritance flags, rather than POSIX ACL's single directory-wide default-ACL-and-mask approach. The practical consequence: NFSv4 ACLs can express access-control policies POSIX ACLs simply cannot, which is exactly why translating FROM a POSIX-ACL'd filesystem TO NFSv4 semantics (when serving that data via NFSv4) isn't a trivial, lossless mapping — the richer model has no equivalent gap to map back INTO on the POSIX side.

**4. A Linux server with POSIX-ACL'd data is being served to clients via NFSv4, and a client reports permission behavior that doesn't quite match what `getfacl` shows on the server. What's your hypothesis, and what would you check?**

My hypothesis is that the NFSv4 server is translating the underlying POSIX ACL into NFSv4 ACL semantics for the client, and something about that translation doesn't perfectly preserve the original POSIX ACL's intended behavior — given that NFSv4's richer model doesn't map onto POSIX ACLs in a fully lossless, one-to-one way, some nuance of the original permission intent can shift during that translation. I'd check the ACTUAL NFSv4 ACL the client is seeing/receiving (using NFSv4-native tooling like `nfs4_getfacl` from a client, or server-side tooling that shows the translated representation) and compare it directly against the source POSIX ACL (`getfacl` on the server) to identify specifically which entries or nuances differ between the two representations, rather than assuming the server-side `getfacl` output is automatically an accurate preview of what NFSv4 clients will actually experience.

**5. Why might a SysAdmin choose to use `getfattr`/`setfattr` directly instead of `getfacl`/`setfacl`, even when working specifically with ACL-related data?**

`getfattr`/`setfattr` operate at the raw extended-attribute level and can be useful for scripting/automation scenarios that need to handle xattrs generically — for example, a backup or migration script that needs to preserve or inspect ALL extended attributes on a file (SELinux context, custom application metadata, AND ACLs) uniformly, without needing separate, tool-specific logic for each different xattr category. That said, for ACL-SPECIFIC work, `setfacl`/`getfacl` remain the strongly preferred, purpose-built tools in almost every practical case — they understand and correctly format the ACL-specific binary encoding into human-readable form and handle ACL-specific concerns (like mask recalculation) correctly, which raw `setfattr` manipulation of the underlying binary xattr value would not handle safely or correctly without deep, unnecessary understanding of the exact on-disk ACL binary format.

---

**End of Day 5.** You should now be able to identify exactly when standard Unix permissions run out of expressive power and reach for ACLs correctly, build default-ACL-based shared directories that don't depend on individual umask discipline, reason precisely about the mask entry's ceiling behavior (including the `chmod`-rewrites-the-mask interaction most admins never learn until it bites them), avoid the ACL-preservation pitfalls that plague real backup/migration workflows, and — as the advanced capstone — explain ACLs as one specific application of the broader extended-attribute mechanism, including how POSIX and NFSv4 ACLs are genuinely different models rather than the same thing wearing a different label.

Proceed to **Day 6 — Firewalld & Network Security** next.
