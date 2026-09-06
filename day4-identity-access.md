# Day 4 — Identity & Access (Full Study Edition, with Answers)

**Estimated time: 5–6 hours**

Today covers the full identity and permission-bit stack a product-company interviewer will actually probe: user/group lifecycle management, password aging and account lockout, sudo/sudoers configuration, and then a genuinely deep pass on UMASK and the SUID/SGID/sticky-bit trio — taken from "what is it" all the way to "how would you audit an entire filesystem for a dangerous one." These special permission bits get asked about constantly at the product-company level specifically because they're a real, live attack surface, not just RHCSA trivia — interviewers want to know you understand the security implications, not just the chmod syntax.

---

## Topic 1: User & Group Management Fundamentals

### Quick Review
- `useradd -m -s /bin/bash -c "Full Name"` creates a user with a home directory and shell.
- UID ranges matter: system accounts typically `<1000`, regular users `≥1000` (governed by `/etc/login.defs`).
- `usermod -aG` **appends** to groups — forgetting `-a` **overwrites** all existing supplementary groups.
- `/etc/skel/` is the template copied into every new home directory — anything dropped here becomes org-wide policy.

### Quick Learning

User management commands are simple to run and easy to get subtly wrong in ways that don't show an error — `usermod -G` without `-a` is the single most common real-world mistake, silently stripping a user out of every group except the one just specified, often breaking their access to shared resources without any error message at the time. Interviewers ask about this specifically because it's a genuine "have you actually operated this in production" filter — someone who's only read the man page is far more likely to have never hit this trap.

**UID/GID ranges and where a new user's identity actually comes from:**
```
  /etc/login.defs
  ────────────────
  UID_MIN   1000      ◄── useradd picks the next free UID at/above this
  UID_MAX   60000
  GID_MIN   1000
  GID_MAX   60000

     0-999:  system/service accounts (root=0, daemon accounts, etc.)
             — useradd won't allocate here automatically
   1000+:    regular human user accounts — this is where YOUR
             useradd command lands unless you override with -u

  useradd -m -s /bin/bash newuser
       │
       ▼
  1. Picks next free UID ≥ UID_MIN from /etc/login.defs
  2. Creates matching private group (GID = same number, if
     USERGROUP scheme is in effect) OR uses a shared default group
  3. Copies /etc/skel/* into the new home directory
  4. Writes entries to /etc/passwd, /etc/shadow, /etc/group
```

### Implementation (Learn by Applying)

**Scenario:** Onboard a new contractor with a hard account-expiry date, correct supplementary group membership, and a customized skeleton so every new hire automatically gets the same baseline shell config — then deliberately reproduce and diagnose the classic `-aG` mistake.

```bash
# Customize the org-wide skeleton BEFORE creating new users, so it applies automatically
echo 'umask 027' >> /etc/skel/.bashrc
mkdir -p /etc/skel/.config

# Create the contractor with a hard expiry date and initial group memberships
useradd -m -s /bin/bash -c "Jane Contractor" -G developers,vpn-users -e 2026-12-31 jcontractor
passwd jcontractor
chage -l jcontractor

id jcontractor
groups jcontractor
```

Now reproduce the classic mistake and diagnose it:
```bash
usermod -G onlythisgroup jcontractor    # WRONG — no -a, this OVERWRITES supplementary groups
groups jcontractor                       # developers and vpn-users are now GONE — no error was ever shown

# Fix it correctly
usermod -aG developers,vpn-users onlythisgroup jcontractor 2>/dev/null
usermod -G onlythisgroup,developers,vpn-users jcontractor   # or explicitly list ALL desired groups if not using -a
groups jcontractor
```

### Interview Questions — with Answers

**1. A user calls in saying they suddenly can't access a shared project folder they've used for months, and nothing about the folder's permissions has changed. What's your first hypothesis and how do you confirm it?**

My first hypothesis is that someone ran `usermod -G` (without `-a`) on this account recently, overwriting their supplementary group memberships and silently removing them from whatever group actually grants access to that shared folder — this produces exactly this symptom (no error at the time, folder permissions genuinely unchanged, user simply no longer has the group membership needed) and is an extremely common real-world mistake. I'd confirm with `groups <username>` or `id <username>` to see their current group memberships and compare against what the shared folder's group ownership (`ls -ld /path/to/folder`) actually requires — if the required group is missing from their list, that confirms it, and checking `/var/log/secure` or admin change history for a recent `usermod` invocation on this account would confirm who/when.

**2. Why does `/etc/login.defs` only affect NEW users, not existing accounts — walk me through a concrete scenario where this distinction matters in an interview or real incident.**

`/etc/login.defs` defines default values that commands like `useradd` and `chage` consult at the moment they're creating a NEW account or when defaults are explicitly applied — it's not a live policy engine that continuously enforces settings against every existing account. A concrete scenario: a security team updates `PASS_MAX_DAYS` in `/etc/login.defs` from 90 to 60, expecting this to tighten password rotation policy across the organization — but every existing user account keeps whatever expiry value was already written into their individual `/etc/shadow` entry at the time their account was created or last had `chage` explicitly run against them, meaning the policy change has zero effect on anyone already onboarded until someone explicitly runs `chage -M 60` against each existing account (or a script does so in bulk).

**3. What's the practical difference between locking an account with `usermod -L` and setting its shell to `/sbin/nologin`? When would you use one over the other?**

`usermod -L` prepends a `!` to the password hash in `/etc/shadow`, which specifically blocks *password-based* authentication — but it does NOT block SSH key-based login, since key auth doesn't consult the password hash at all, meaning a locked account can often still log in via SSH keys unless key auth is separately disabled for that account. Setting the shell to `/sbin/nologin` blocks interactive shell access entirely regardless of authentication method — the login succeeds at the auth layer, but the user is immediately dropped since there's no usable shell to hand them. I'd use `-L` for a temporary suspension where you specifically want to block password login only (e.g., during a password reset process), and `/sbin/nologin` for accounts that should never get an interactive shell at all — service accounts, or genuinely offboarded users where you want a harder block than password-only locking provides.

**4. You're onboarding many new hires and want every one of them to automatically get a specific umask setting and a pre-populated directory structure without manually configuring each account. How do you achieve that?**

I'd customize `/etc/skel/` — anything placed there (a `.bashrc` with a specific `umask` line, pre-created subdirectories, config file templates) gets copied automatically into every new user's home directory the moment `useradd -m` creates it, with zero per-user manual steps required going forward. This is exactly the org-wide-policy-via-template pattern: edit `/etc/skel/.bashrc` once, and every subsequently created account inherits that setting automatically, rather than needing a separate onboarding script to configure each account individually after the fact.

**5. What's the actual difference between `userdel username` and `userdel -r username`, and why does forgetting `-r` create a slow-burning cleanup problem months later?**

Plain `userdel` removes the account entries from `/etc/passwd`, `/etc/shadow`, and `/etc/group`, but leaves the user's home directory and mail spool completely intact on disk. `userdel -r` additionally removes the home directory and mail spool as part of the same operation. Forgetting `-r` repeatedly across many offboarded users over time leaves an accumulating pile of orphaned home directories — consuming disk space with no corresponding account, and eventually becoming a genuine "who owns this and can we delete it" archaeology problem when someone finally needs to reclaim that space, since the original UID may no longer resolve to any username at all (`ls -l` would show a bare numeric UID instead of a name).

---

## Topic 2: Password Aging, Account Locking & PAM Basics

### Quick Review
- `chage` controls per-user password aging: max age, min age, warning period, inactive-after-expiry grace period.
- `faillock` (RHEL 8+, replaces `pam_tally2`) tracks and can auto-lock accounts after repeated failed login attempts.
- PAM (Pluggable Authentication Modules) is the actual framework enforcing auth policy — `chage`/`faillock` are front-ends configuring PAM-consulted state, not separate mechanisms.
- `passwd -S` gives a fast one-line status check (locked/aging state) without the full `chage -l` detail dump.

### Quick Learning

Password aging and account lockout aren't separate ad-hoc features bolted onto Linux — they're all expressions of the underlying PAM stack, which is the actual framework consulted at every authentication attempt to decide "should this login succeed." Understanding that `chage` writes aging fields into `/etc/shadow` which `pam_unix.so` reads, and that `faillock` maintains its own state which `pam_faillock.so` reads, is what lets you reason about *why* a particular login is failing when the surface-level symptom ("user says they can't log in") could stem from several genuinely different underlying causes.

**Where a login decision actually gets made:**
```
  User attempts SSH login
        │
        ▼
  sshd hands off authentication to PAM
        │
        ▼
  /etc/pam.d/sshd  (the actual PAM STACK for this service)
        │
        ├─▶ pam_faillock.so   ── checks: has this account had too
        │                          many recent failed attempts?
        │                          (state in /var/run/faillock/)
        │
        ├─▶ pam_unix.so       ── checks: password hash match,
        │                          AND /etc/shadow aging fields
        │                          (expired? inactive-too-long?)
        │
        └─▶ (other modules, e.g. pam_sepermit, pam_selinux, etc.)

  ANY module in the stack can deny the login — "wrong password"
  might actually be masking "account is faillock-locked" or
  "password aged out" depending on WHICH module actually said no.
```

### Implementation (Learn by Applying)

**Scenario:** Configure a compliant password-aging policy for a production account, then reproduce and recover from a faillock-triggered lockout — a very common real support ticket that's often misdiagnosed as "forgot their password."

```bash
chage -M 90 -m 7 -W 7 -I 14 produser
chage -l produser
passwd -S produser

# Configure faillock policy: lock after 5 failed attempts within 15 minutes, for 30 minutes
authselect current                          # Confirm current profile before editing PAM directly
# In practice, faillock deny/unlock_time is typically set via authselect custom profiles or
# directly in /etc/security/faillock.conf on RHEL 8/9:
grep -E "^deny|^unlock_time" /etc/security/faillock.conf
```

Reproduce a lockout deliberately:
```bash
for i in 1 2 3 4 5 6; do
  su - produser -c "true" <<< "wrongpassword" 2>/dev/null
done

faillock --user produser
```

Diagnose and recover:
```bash
passwd -S produser        # Might still show password as valid — the block is faillock, not password aging
faillock --user produser  # Shows the actual failed-attempt count and whether it's currently locked

faillock --user produser --reset    # Clear the lockout
```

### Interview Questions — with Answers

**1. A user insists they're typing their password correctly, but login keeps failing. `passwd -S` shows the password itself is valid and not expired. What else would you check, and why might `chage` alone not tell the full story?**

I'd check `faillock --user <username>` specifically — `chage`/`passwd -S` only report on password *aging* state (expired, inactive, etc.), which is an entirely separate mechanism from failed-attempt lockout tracked by `pam_faillock`. It's entirely possible for a password to be perfectly valid and unexpired while the account is simultaneously locked out due to accumulated recent failed attempts (perhaps from an old cached credential on their phone/laptop retrying automatically in the background without their knowledge) — `chage`/`passwd -S` would show everything looking fine because they're checking a completely different subsystem than the one actually blocking the login.

**2. Explain the difference between `PASS_MAX_DAYS`, `PASS_MIN_DAYS`, and `PASS_WARN_AGE` (or their `chage` equivalents `-M`, `-m`, `-W`) — specifically, why does a MINIMUM age setting exist at all? Isn't more frequent password changing always safer?**

`-M`/`PASS_MAX_DAYS` is the maximum number of days a password can be used before it's forcibly expired. `-m`/`PASS_MIN_DAYS` is the MINIMUM number of days that must pass before the user is allowed to change their password again. `-W`/`PASS_WARN_AGE` is how many days before expiry the user starts seeing a warning at login. The minimum age exists specifically to prevent a common workaround to forced rotation policies: without it, a user facing a mandatory password change can simply change their password, then immediately change it right back to their original password, technically satisfying the rotation requirement while defeating its actual security purpose — a minimum age prevents that same-day round-trip, forcing the new password to actually stay in effect for a meaningful period.

**3. What's the actual mechanism behind `faillock`, and why is it a PAM-based solution rather than something built into `/etc/shadow` the way password aging is?**

`faillock` works by having the `pam_faillock.so` module maintain its own separate state (tracked per-user, typically under `/var/run/faillock/`) recording timestamps of recent failed authentication attempts, and consulting that state on every subsequent attempt to decide whether the configured failure threshold within the configured time window has been exceeded, triggering a temporary lock if so. It's PAM-based rather than shadow-based because failed-login tracking is fundamentally about the AUTHENTICATION ATTEMPT process itself — a real-time, per-attempt decision — which is exactly what PAM's modular stack architecture is designed to intercept and evaluate, whereas `/etc/shadow`'s aging fields describe static properties of the password itself (when it was set, when it should expire) that don't need to be evaluated at the granularity of "this specific attempt, right now."

**4. If you needed to see exactly which PAM module is responsible for a specific service's authentication decisions (say, `sshd`), where would you look, and what would you expect to find?**

I'd look at `/etc/pam.d/sshd` — each PAM-aware service has its own configuration file in `/etc/pam.d/` defining the ordered stack of modules consulted for that service's authentication, account, session, and password management phases. I'd expect to see lines referencing modules like `pam_faillock.so` (failed-attempt tracking), `pam_unix.so` (traditional Unix password/shadow-based auth), possibly `pam_sss.so` if the system is integrated with SSSD/AD, and various control flags (`required`, `sufficient`, `requisite`) determining how failures/successes in each module affect the overall stack's decision — reading this file directly is the authoritative way to understand exactly what's being checked for that specific service, rather than assuming based on general Linux auth knowledge.

**5. You need to enforce a policy where accounts are automatically disabled (not just password-expired, but genuinely disabled) if they haven't been used to log in for 30 days. How would you approach this, since it's not something `chage` alone directly tracks?**

`chage -I` (inactive) is actually the closest built-in mechanism — it specifies how many days AFTER password expiry the account can go without a password change before being automatically locked, but that's tied to password expiry timing, not last-login timing specifically. For genuine last-login-based disabling, I'd build a scheduled script (via a systemd timer, per Day 3) that runs `lastlog` to check each account's actual last login timestamp, and for any account exceeding the 30-day threshold, explicitly runs `usermod -L` (or sets an expiry date via `chage -E`) against it — since there's no single built-in Linux mechanism that natively enforces "disable based on inactivity specifically," combining `lastlog`'s reporting with a scripted enforcement action is the practical, standard approach.

---

## Topic 3: sudo & sudoers Deep Dive

### Quick Review
- Always edit via `visudo` — it syntax-checks before saving, preventing a broken sudoers file from locking everyone out.
- `/etc/sudoers.d/` drop-in files are the maintainable pattern — survive updates, easy to manage per-team.
- Scope `NOPASSWD` to specific commands, never blanket `NOPASSWD: ALL` in production.
- Rule ORDER matters — sudoers evaluates top-to-bottom, and a LATER matching rule can override an earlier one.

### Quick Learning

sudo's real complexity isn't the basic `%group ALL=(ALL) ALL` syntax — it's understanding rule precedence and scoping well enough to grant exactly the access needed and nothing more, which is precisely the security-conscious mindset a product-company interviewer is screening for. A blanket `NOPASSWD: ALL` for a whole team is the equivalent of handing out root's password to everyone in that team — functional, but a real audit finding waiting to happen.

**Sudoers evaluation — later matching rules win:**
```
  /etc/sudoers.d/10-developers
  ────────────────────────────
  %developers ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart app.service

  /etc/sudoers.d/20-developers-override
  ────────────────────────────────────
  %developers ALL=(ALL) NOPASSWD: ALL      <- loaded AFTER 10-*, alphabetically

  Result: developers actually get FULL NOPASSWD sudo, because sudoers
  files are processed in a defined order (visudo/sudo reads
  /etc/sudoers.d/* typically in filename order) and a LATER, more
  permissive matching rule for the same user/group can effectively
  override the earlier, more restrictive intent — this is why
  naming/ordering drop-in files carefully, and reviewing the FULL
  effective policy (not just the file you just wrote), matters.
```

### Implementation (Learn by Applying)

**Scenario:** Grant the `developers` group narrowly-scoped, passwordless permission to restart exactly one application service — nothing else — using the drop-in pattern, then deliberately create an ordering conflict to see how it actually resolves.

```bash
visudo -f /etc/sudoers.d/10-developers
```
Add:
```
%developers ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart app.service, /usr/bin/systemctl status app.service
```
```bash
visudo -c                          # Check syntax of the whole sudoers configuration, not just this file
chmod 440 /etc/sudoers.d/10-developers   # sudoers drop-ins require these exact permissions

# Test as a member of developers
su - someuser -c "sudo systemctl restart app.service"   # Should succeed with no password prompt
su - someuser -c "sudo systemctl restart sshd"           # Should be DENIED — scoped correctly
```

Now demonstrate the ordering trap:
```bash
visudo -f /etc/sudoers.d/20-developers-broad
```
Add:
```
%developers ALL=(ALL) NOPASSWD: ALL
```
```bash
sudo -l -U someuser    # Shows the FULL effective ruleset for this user — review this, not just individual files
```
Remove the overly broad file immediately after confirming the lesson:
```bash
rm /etc/sudoers.d/20-developers-broad
```

### Interview Questions — with Answers

**1. Why should you always use `visudo` instead of editing `/etc/sudoers` (or files in `/etc/sudoers.d/`) directly with a normal text editor?**

`visudo` performs a syntax check on the file before allowing the save to actually take effect — if you've introduced a syntax error, it warns you and gives you the choice to fix it or discard the change, rather than saving a broken file. Editing directly with `vi`/`nano` and saving a syntactically broken sudoers file can lock out EVERYONE's ability to use `sudo` system-wide, including yourself, potentially requiring single-user/rescue mode recovery just to fix a typo — `visudo`'s validation step exists specifically to prevent that scenario, and skipping it for "just a quick edit" is a real, avoidable production risk.

**2. Explain why `NOPASSWD: ALL` for an entire team is considered a security anti-pattern, even if that team is trusted. What's the more scoped alternative and its tradeoff?**

Blanket `NOPASSWD: ALL` grants every member of that team the ability to run absolutely any command as root with zero authentication friction and zero command-specific restriction — functionally equivalent to giving them the root password, removing the audit trail's ability to distinguish "this specific, expected administrative action" from "literally anything," and removing the password-reentry friction that can sometimes catch an accidental or automated action before it executes. The more scoped alternative is explicitly listing the specific commands a role actually needs (as in the lab), which requires more upfront thought about exactly what each team genuinely needs to do, and requires updating the sudoers entry whenever their legitimate needs expand — a real ongoing maintenance cost, but one that trades a small amount of admin overhead for a dramatically smaller blast radius if an account is compromised or a command is run by mistake.

**3. You've written a sudoers rule that looks correct, but a user reports having MORE access than you intended. What's your troubleshooting approach, given that sudoers files can span multiple drop-ins?**

I would not just re-read the specific file I wrote — I'd run `sudo -l -U <username>` to see the FULL, effective, resolved ruleset sudo actually applies for that user, accounting for every file in `/etc/sudoers.d/` and the main sudoers file combined, in their actual evaluation order. This is essential because, as demonstrated in the lab, a separate, later-loaded drop-in file can grant broader access than the one you're focused on, and the only reliable way to see the true effective permission set is asking sudo itself to report it, rather than manually reasoning through multiple files and their processing order in your head.

**4. What's the difference between `%groupname` and a plain username in a sudoers entry, and why might using groups be operationally preferable for a growing team?**

`%groupname` matches any user who is a member of that Unix group, while a plain username matches only that one specific account — group-based rules automatically apply to anyone added to the group in the future, with zero sudoers file changes required, whereas per-user entries require an explicit sudoers edit every time someone joins or leaves the relevant role. For a growing or changing team, group-based rules are operationally preferable because onboarding/offboarding becomes purely a group-membership change (`usermod -aG`/`gpasswd -d`) rather than requiring a sudoers file edit each time — reducing the chance of someone being granted excess access and forgotten about, or a departing team member's sudo rule being overlooked during offboarding.

**5. A command needs to run with sudo, but the sudoers rule specifies the exact full path to the binary. A user reports "command not found" even though the binary genuinely exists and works fine without sudo. What's the likely mismatch, and how do you find it?**

The most likely mismatch is that the path specified in the sudoers rule doesn't exactly match the actual, real path to the binary the user is invoking — perhaps due to a symlink, a different installation location than assumed, or the user's shell resolving the command via `$PATH` to a different binary than the one explicitly whitelisted in sudoers (sudo intentionally does NOT use the invoking user's `$PATH` to resolve the command against the sudoers rule, specifically to prevent path-based tricks). I'd find the actual real path with `which <command>` or `readlink -f $(which <command>)` as that specific user would resolve it, and compare that exact string, character for character, against what's written in the sudoers rule — a subtle difference (e.g., `/usr/bin/systemctl` in sudoers versus the binary actually living at `/usr/local/bin/systemctl` via a different installation) is a common, easy-to-miss root cause.

---

## Topic 4: UMASK — Basic to Advanced

### Quick Review
- Default permissions for new files: `666` (rw-rw-rw-); new directories: `777` (rwxrwxrwx) — umask SUBTRACTS from these.
- `umask 022` → files created at `644`, directories at `755`.
- Persistence order: `/etc/profile`/`/etc/bashrc` (system-wide) → `~/.bashrc` (per-user override) → `/etc/login.defs` `UMASK` (affects `useradd`-created home dir permissions specifically).
- umask is a **per-process** inherited setting — a running daemon's umask is fixed at the point it was started, and changing your shell's umask later doesn't retroactively affect it.

### Quick Learning

Basic umask is "subtract this from the default" — advanced umask is understanding that it's a *per-process, inherited* attribute, not a system-wide setting, which explains a whole category of "why does this service create files with different permissions than my shell would" confusion. A systemd service's effective umask comes from its own `UMask=` directive (or the default if unset) — NOT from whatever your interactive shell happens to have configured, because the service was launched by PID 1 in an entirely separate process lineage, not forked from your login shell.

**How umask actually produces a final permission value:**
```
  Default MAXIMUM permissions (before umask is ever applied):
     Files:        rw-rw-rw-   (666)  — files never default to executable
     Directories:  rwxrwxrwx   (777)

  umask 022  means: "REMOVE write from group, REMOVE write from other"
     022 in binary-per-digit:   0  2  2
                                 │  │  │
                                 │  │  └─ other: remove w (bit value 2)
                                 │  └──── group: remove w (bit value 2)
                                 └─────── owner: remove nothing

  New FILE with umask 022:      666 - 022 = 644  (rw-r--r--)
  New DIRECTORY with umask 022: 777 - 022 = 755  (rwxr-xr-x)

  Stricter example — umask 027:
  New FILE:      666 - 027 = 640  (rw-r-----)
  New DIRECTORY: 777 - 027 = 750  (rwxr-x---)
  (removes ALL other-access, and group loses write — common for
   shared-team directories where "other" should see nothing at all)
```

**Where a process's umask actually comes from — inheritance, not a global setting:**
```
  login shell (umask set by /etc/profile, then ~/.bashrc)
        │
        ├─▶ any command YOU run from this shell inherits ITS umask
        │
        ▼
  cron job (umask often defaults to 022 regardless of YOUR shell's umask,
            unless the crontab or script explicitly sets one)

  systemd service (umask comes from the unit's OWN UMask= directive,
                    default 022, completely independent of any
                    user's interactive shell umask, since systemd/PID 1
                    is the parent, not your login shell)
```

### Implementation (Learn by Applying)

**Scenario:** A team's shared project directory needs a stricter default than the system-wide umask, applied ONLY for that team, persistently — and separately, diagnose a real "why do my cron job's output files have different permissions than when I run the script manually" ticket.

```bash
# Confirm current umask and its effect
umask
touch /tmp/testfile_default && ls -l /tmp/testfile_default

umask 027
touch /tmp/testfile_strict && ls -l /tmp/testfile_strict     # Compare: 644 vs 640
```

Apply team-specific umask persistently, without changing it system-wide:
```bash
echo "umask 027" >> /home/teamlead/.bashrc    # Per-user persistent override
# For a whole GROUP of users sharing a role, a common pattern is a shared login script
# sourced via /etc/bashrc conditionally based on group membership:
cat >> /etc/bashrc << 'EOF'
if id -nG "$USER" | grep -qw "finance-team"; then
    umask 027
fi
EOF
```

Now reproduce and diagnose the cron/manual-run permission mismatch:
```bash
cat > /usr/local/bin/generate_report.sh << 'EOF'
#!/bin/bash
touch /tmp/report_output.txt
EOF
chmod +x /usr/local/bin/generate_report.sh

# Manual run, using YOUR shell's umask
/usr/local/bin/generate_report.sh
ls -l /tmp/report_output.txt
rm /tmp/report_output.txt

# Simulate cron's typical environment — explicitly different umask, since cron doesn't
# source your interactive shell's .bashrc at all
(umask 022; /usr/local/bin/generate_report.sh)
ls -l /tmp/report_output.txt
```
The fix, if consistent permissions matter for the generated file regardless of caller: set `umask` EXPLICITLY inside the script itself, rather than relying on inherited environment from whatever invoked it.
```bash
sed -i '2i umask 077' /usr/local/bin/generate_report.sh
cat /usr/local/bin/generate_report.sh
```

### Interview Questions — with Answers

**1. Explain, mechanically, how a umask value of `022` produces a resulting permission of `755` for a new directory, starting from first principles.**

Directories start from a maximum default of `777` (rwxrwxrwx) before any umask is applied. umask specifies which permission BITS to REMOVE from that maximum, expressed per-digit for owner/group/other: `022` means remove nothing from owner (0), remove write (value 2) from group, and remove write (value 2) from other. Subtracting bitwise: owner keeps `rwx` (7), group loses write leaving `r-x` (5), other loses write leaving `r-x` (5) — giving `755` overall, which matches the commonly-seen default directory permission on most systems using the standard `022` umask.

**2. Why do newly created FILES never get the executable bit set by default, even with a very permissive umask like `000`?**

The starting maximum for files is `666` (rw-rw-rw-), not `777` — files categorically never include the execute bit in their default maximum permission set, regardless of umask, because umask can only REMOVE bits from that starting maximum, never ADD bits that weren't in the ceiling to begin with. This is a deliberate security-oriented design choice: a newly created file (say, output from a text editor or a script redirecting output) shouldn't automatically become executable just because a permissive umask is in effect — making a file executable is expected to be an explicit, separate action (`chmod +x`), never an accidental side effect of umask alone.

**3. A cron job's generated files have different permissions than when the same script is run manually by the same user from their interactive shell. What's actually going on, and what's the correct fix?**

Cron doesn't execute jobs within the context of the user's interactive login shell — it runs them in a minimal environment that does NOT source `~/.bashrc` (where a user's personal umask customization typically lives), meaning cron jobs typically run with whatever umask is set at a more basic level (often the system default `022`) rather than picking up any per-user shell customization the person is used to seeing when running things manually. The correct, robust fix is to set `umask` explicitly INSIDE the script itself, at the top, so the script's file-creation behavior is deterministic and correct regardless of what environment or umask the CALLER happens to have — relying on inherited umask from whatever invokes the script is fragile precisely because different invocation contexts (interactive shell, cron, systemd timer) can have genuinely different inherited umask values.

**4. If you set `umask 022` in your current interactive shell session and it works as expected, but a long-running daemon that was already running before you made that change still creates files with the OLD, more permissive permissions — why doesn't your umask change affect it?**

umask is a per-process attribute, inherited by a process from its PARENT at the moment it was forked/exec'd — it's not a live, global system setting that every running process continuously re-reads. A daemon that was already running before your shell's umask change inherited whatever umask was in effect from ITS parent process (likely PID 1/systemd, or an init script) at the time IT started, and that inherited value is now fixed for that process's entire lifetime — changing your own shell's umask afterward has zero retroactive effect on a process that already started with a different value baked in. To actually change that daemon's file-creation behavior, you'd need to change its own startup configuration (e.g., a systemd `UMask=` directive) and restart it, not just change your unrelated interactive shell's umask.

**5. You need a shared team directory where every new file/subdirectory automatically gets restrictive `other`-level permissions (umask-style), but you can't guarantee every team member's personal shell umask is configured correctly. What's a more robust solution than relying on umask alone?**

Relying purely on each user's personal umask being correctly configured is fragile — it depends on every team member's individual shell config being right, which you can't fully control or guarantee, especially as people join with their own dotfile setups. A more robust approach is setting the SGID bit on the shared directory (`chmod g+s /shared/teamdir`) combined with a default ACL (`setfacl -m d:o::-  /shared/teamdir`, or more completely, default ACLs granting exactly the group/user permissions needed) — SGID ensures new files inherit the DIRECTORY's group rather than the creator's primary group (covered in depth in Topic 5), and a default ACL can enforce specific permission bits on new files/subdirectories regardless of the creating user's personal umask, since the ACL's inheritance takes effect at the directory level rather than depending on each individual user's shell environment being correctly configured.

---

## Topic 5: SUID, SGID & Sticky Bit — Basic to Advanced

### Quick Review
- **SUID** (`chmod u+s`, shown as `s` in the owner-execute position) — an executable runs with the **file owner's** privileges, not the invoking user's.
- **SGID** (`chmod g+s`) on a file — runs with the **file's group** privileges; on a **directory** — new files/subdirs inherit the directory's group automatically.
- **Sticky bit** (`chmod +t`, shown as `t` in the other-execute position) — on a directory, users can only delete their OWN files, even with directory write access.
- SUID root binaries are a genuine, actively-exploited attack surface — auditing for unexpected ones is a real security practice, not paranoia.

### Quick Learning

These three bits occupy the same conceptual "extra" position beyond the standard rwx triplet, but they mean completely different things depending on whether they're set on a file versus a directory, and conflating them is the most common way candidates lose credibility on this topic. SUID makes sense on files (run-as-owner); it has no meaningful standard effect on directories. SGID means something different depending on file-vs-directory — run-as-group for a file, but new-content-inherits-group for a directory, which is the specific behavior that makes it so useful for shared team folders. The sticky bit is essentially directory-only in practice (its historical file-level meaning is obsolete on modern Linux) and solves one specific problem: preventing users from deleting each other's files in a shared, world-writable location.

**The permission bit layout, all four "extra" positions at once:**
```
  Standard rwx triplet, PLUS the special bit encoded in the same digits:

      SUID  SGID  sticky
       │     │      │
       ▼     ▼      ▼
     ---s ---s ---t     <- capital S/T if the underlying x bit is ABSENT
      (owner) (group) (other)

  Octal representation — the special bits are a FOURTH leading digit:
      4000 = SUID
      2000 = SGID
      1000 = sticky bit

  Example: chmod 4755 /usr/bin/passwd
      4    7     5     5
      │    │     │     │
      │    owner group other
      SUID (rwx) (r-x) (r-x)

  ls -l output:  -rwsr-xr-x   1 root root  ... /usr/bin/passwd
                     ▲
                     lowercase 's' = SUID bit SET, and owner-execute
                     is ALSO set (if owner-execute were OFF, you'd
                     see a capital 'S' instead — a common gotcha:
                     a capital S/T often indicates a MISCONFIGURED
                     special bit with no matching execute permission,
                     which usually has no effect at all)
```

**SUID in action — why `passwd` needs it:**
```
  Regular user runs:  passwd
        │
        ▼
  Without SUID: passwd runs with the INVOKING user's privileges
                ⇒ cannot write to /etc/shadow (root-owned, 000/600)
                ⇒ users could never change their own password

  WITH SUID (owner = root):  passwd runs with ROOT's privileges
                ⇒ CAN write to /etc/shadow, but ONLY through the
                  passwd binary's own controlled logic (verify old
                  password, enforce complexity, THEN write) —
                  the SUID bit grants the ELEVATED PRIVILEGE, the
                  program's own code is what constrains HOW that
                  privilege gets used
```

**SGID on a directory — the shared-team-folder use case:**
```
  WITHOUT SGID                              WITH SGID (chmod g+s /shared)
  ────────────                              ──────────────────────────
  /shared  (group: devteam)                 /shared  (group: devteam, SGID set)
     │                                          │
     ▼ alice creates newfile.txt                ▼ alice creates newfile.txt
  newfile.txt group = alice's PRIMARY           newfile.txt group = devteam
              group (e.g. "alice")                          (inherited from
                                                               the DIRECTORY,
  ⇒ bob (also in devteam, but with a                          not from alice's
    DIFFERENT primary group) may not                          own primary group)
    have group access to alice's file
    at all, depending on group setup          ⇒ bob (also in devteam) automatically
                                                 has the SAME group-level access
                                                 to every file anyone creates here
```

**Sticky bit — the classic `/tmp` scenario:**
```
  /tmp  permissions: drwxrwxrwt   (1777)
                              │
                              └─ sticky bit ('t') — EVERYONE can write
                                 (create files) here, but...

  alice creates /tmp/alice_file.txt
  bob has normal WRITE access to /tmp (it's world-writable)
        │
        ▼
  WITHOUT sticky bit: bob's write access to the /tmp DIRECTORY
                       would let him DELETE alice_file.txt too,
                       since directory write permission normally
                       governs delete rights for contents

  WITH sticky bit:     bob can create HIS OWN files in /tmp fine,
                       but CANNOT delete alice_file.txt — only
                       alice (the file's owner) or root can
```

### Implementation (Learn by Applying)

**Scenario:** Set up a shared team directory with SGID for automatic group inheritance, verify SUID behavior on a real system binary, and configure a sticky-bit shared upload directory — then, critically, audit the system for existing SUID root binaries, the security-relevant capstone of this topic.

```bash
# SGID shared team directory
groupadd devteam
mkdir /shared
chgrp devteam /shared
chmod 2775 /shared              # 2 = SGID, 775 = rwxrwxr-x
ls -ld /shared                  # Confirm: drwxrwsr-x  (lowercase s in group-execute position)

usermod -aG devteam alice
usermod -aG devteam bob
su - alice -c "touch /shared/alice_file.txt"
ls -l /shared/alice_file.txt    # Confirm group is devteam, NOT alice's personal primary group
```

Examine SUID on a real system binary and understand what it actually protects:
```bash
ls -l /usr/bin/passwd            # -rwsr-xr-x — SUID set, owned by root
ls -l /etc/shadow                # ----------  root root — NOT even root's own group can read this directly

# Prove the mechanism: a regular user CAN change their own password (via the SUID binary's controlled logic)
# but CANNOT read /etc/shadow directly themselves
su - alice -c "cat /etc/shadow" 2>&1     # Permission denied
su - alice -c "passwd" <<< $'newpass123\nnewpass123'  # Succeeds — SUID lets the binary itself write to shadow
```

Sticky bit on a shared upload directory:
```bash
mkdir /uploads
chmod 1777 /uploads              # 1 = sticky, 777 = rwxrwxrwx
ls -ld /uploads                  # drwxrwxrwt

su - alice -c "touch /uploads/alice_upload.txt"
su - bob -c "rm /uploads/alice_upload.txt" 2>&1    # Denied — sticky bit protects it even though /uploads itself is world-writable
```

**The advanced/security capstone — auditing the whole filesystem for SUID root binaries:**
```bash
find / -perm -4000 -type f 2>/dev/null                       # All SUID files, system-wide
find / -perm -4000 -user root -type f 2>/dev/null            # Specifically SUID files owned by root — the real risk category
find / -perm -2000 -type f 2>/dev/null                       # All SGID files, for the same audit purpose

# Compare against a known-good baseline (e.g., right after a fresh minimal install) to spot anything ADDED later
find / -perm -4000 -type f 2>/dev/null | sort > /root/suid_baseline_current.txt
diff /root/suid_baseline_original.txt /root/suid_baseline_current.txt 2>/dev/null
```

### Interview Questions — with Answers

**1. Explain exactly why `/usr/bin/passwd` needs the SUID bit, and what would happen if it didn't have it. Walk through the actual mechanism, not just "it needs elevated privileges."**

`/etc/shadow`, which stores password hashes, is deliberately owned by root with permissions that block even read access for any non-root user — this is intentional, since password hashes are sensitive. When a regular user runs `passwd` to change their own password, that program needs to WRITE to `/etc/shadow`, an operation a regular user's own privilege level could never perform directly. The SUID bit, with `passwd` owned by root, means the program executes with ROOT's effective privileges for the duration of its run — regardless of which regular user invoked it — allowing it to write to `/etc/shadow`, but ONLY through the program's own internal logic (verify the user knows their current password, enforce complexity rules, THEN write the new hash). Without SUID, the `passwd` binary would run with the invoking user's own limited privileges and would fail with a permission error the moment it tried to write to `/etc/shadow` — meaning users could never change their own passwords through the normal command at all.

**2. Why is an unexpected, unaccounted-for SUID root binary considered a serious security finding, even if the binary itself "looks legitimate"? Walk through the actual attack scenario.**

Any SUID root binary runs with FULL root privileges regardless of who invokes it — if that binary has ANY exploitable flaw (a buffer overflow, a way to be tricked into executing arbitrary commands, an unsafe way of handling input or environment variables), a completely unprivileged local user can potentially leverage that flaw to gain full root access, turning a minor local-access compromise into complete system compromise. An "unexpected" SUID binary specifically is concerning because it suggests either a misconfiguration (someone set SUID on something that never needed it, widening the attack surface for no operational benefit) or, in a worse case, it could be a deliberately planted backdoor by an attacker who's already gained some level of access and is establishing a privilege-escalation path for later — a script or copied shell binary with SUID root set is a classic, simple persistence/escalation technique once initial access is achieved.

**3. What's the practical difference between SGID on a FILE versus SGID on a DIRECTORY? Many candidates only know one of these — demonstrate you know both.**

SGID on a FILE (an executable) causes that program to run with the privileges of the file's GROUP owner, rather than the invoking user's own primary group — analogous to SUID, but substituting group identity for user identity; this is less commonly used in practice than SUID but follows the same "run with a different, elevated identity than the invoker" principle. SGID on a DIRECTORY is a completely different behavior: any new file or subdirectory created inside that directory automatically inherits the DIRECTORY's group ownership, rather than the creating user's own primary group — this is the mechanism that makes shared team directories work sensibly, ensuring every file dropped there by any team member ends up with consistent group ownership automatically, rather than each file having whatever the individual creator's personal primary group happened to be.

**4. A shared directory has the sticky bit set, and a user reports they can't delete a file they're SURE they created. What would you check to resolve the apparent contradiction?**

I'd check the ACTUAL ownership of the specific file in question with `ls -l` — the sticky bit's protection is based on genuine file ownership (as recorded in the filesystem), not the user's memory or assumption of having created it; it's entirely possible the file was actually created by someone else, or by an automated process running as a different user (e.g., a script the user triggered that ran under a service account, rather than their own personal identity), which would fully explain why the sticky bit is correctly preventing deletion despite the user's belief that it's "their" file. I'd also confirm whether the user has root/sudo access being tested incorrectly (sticky bit doesn't restrict root, only regular users), and double check there isn't a separate permissions issue (like an ACL, covered on Day 5) additionally restricting the delete beyond just the sticky bit's normal behavior.

**5. Design an audit process a SysAdmin should run periodically to catch dangerous or unexpected SUID/SGID binaries before they become a security incident. What's the actual workflow, not just the `find` command?**

The workflow starts with establishing a known-good BASELINE, ideally captured right after a fresh, minimal, trusted OS installation, using `find / -perm -4000 -type f` (and the SGID equivalent with `-2000`) to record every legitimately-present SUID/SGID binary at that clean state. Going forward, on a regular schedule (e.g., a weekly systemd timer, tying back to Day 3), I'd re-run the same `find` command and `diff` the current output against the baseline — any NEW entry that wasn't in the original baseline is immediately suspicious and warrants investigation (was this from a legitimate package update that intentionally added a new SUID tool, or something that shouldn't be there at all). Critically, the workflow isn't just running the `find` command once reactively during an incident — it's the recurring, scheduled comparison against a trusted baseline that actually catches something newly and unexpectedly added, which a one-time manual check would never reveal.

---

**End of Day 4.** You should now be able to manage the full user/group lifecycle correctly (including avoiding the classic `-aG` mistake), reason precisely about PAM-driven authentication decisions (distinguishing password-aging failures from faillock lockouts), write scoped and safely-ordered sudoers rules, explain umask as a per-process inherited attribute rather than a global setting, and — the deep capstone of this day — explain and demonstrate SUID/SGID/sticky-bit behavior from first principles through to designing an actual security audit process for unexpected SUID root binaries.

Proceed to **Day 5 — Permissions Deep Dive (ACLs)** next.
