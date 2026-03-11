# Linux-Admin-Project-Phoenix
Linux administration job simulation covering user management, file permissions, log investigation and access control on Ubuntu Linux.

# Linux Administration Job Simulation: Project Phoenix

![Platform](https://img.shields.io/badge/Platform-Ubuntu%20Linux-E95420?style=flat&logo=ubuntu&logoColor=white)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=flat)
![Type](https://img.shields.io/badge/Type-Job%20Simulation-blue?style=flat)

---

## What This Is

This is a hands-on Linux administration simulation I completed over several days. The scenario was built around a fictional project called Project Phoenix where I was brought in as a junior sysadmin to keep the environment stable, investigate failures and lock things down from a security standpoint.

Each day had a different focus. By the end I had gone from basic identity checks to managing user accounts and enforcing access control on a live Linux environment.

---

## Environment

| Field | Detail |
|-------|--------|
| OS | Ubuntu Linux |
| Shell | Bash / Zsh |
| Simulation Platform | LabEx |
| Duration | 3 Days |
| Role | Junior System Administrator |

---

## Day 1: Access and Identity

Before doing anything on the server I needed to know who I was and what I had access to. This sounds basic but it matters. Jumping into a system without knowing your privilege level or group memberships is how you cause damage or miss permission errors that waste time later.

**What I ran:**

```bash
whoami          # confirm current user
id              # check UID, GID and group memberships
groups          # list group names
uname -a        # kernel version and architecture
hostnamectl     # OS summary
cat /etc/os-release  # exact Linux distribution
who             # who is currently logged in
w               # who is logged in and what they are running
last            # recent login history
cat /etc/passwd # all user accounts on the system
cat /etc/group  # all defined groups
getent group sudo  # check who is in the sudo group
top             # CPU and memory usage
uptime          # system load and uptime
df -h           # disk space in readable format
```

**What I learned:**

Knowing the kernel version, who is on the box and how much disk is left gives you a full picture of the system state before you start touching things. The `w` command in particular is more useful than `who` because it shows what each logged-in user is actively running.

---

## Day 2: Log Investigation and File Permissions

Day 2 had two parts. First I investigated why Project Phoenix was failing. Then I was handed a security task to lock down the project directory.

### Log Investigation

The application was throwing failures and no one knew why. I filtered the app log for ERROR entries and saved the output to a report file.

```bash
grep "ERROR" ~/project/logs/app.log > ~/project/error_report.txt
```

Then I moved to the kernel level. The server had unstable boot behavior. I used `dmesg` to pull hardware and kernel logs and searched for errors.

```bash
dmesg | grep -i "error" > ~/project/boot_issues.txt
```

First attempt with `grep "error"` came back empty. Turns out `grep` is case-sensitive by default and the log entries used uppercase. Adding `-i` fixed it and the driver probe error showed up immediately.

Then I compared staging and production configs to find why behavior differed between environments.

```bash
diff staging.conf production.conf > ~/project/config_diff.txt
diff server1_files/ server2_files/
```

The directory diff identified files that were present in one deployment but missing from the other. That gave the dev team a concrete list of what needed to be redeployed.

### File Permissions and Ownership

After the investigation I was tasked with securing the project directory. I transferred ownership to `dev_lead` and the `developers` group then set permissions so only they could access it.

```bash
sudo chown -R dev_lead:developers ~/phoenix_project
sudo chmod 750 ~/phoenix_project
ls -ld ~/phoenix_project
ls -lR ~/phoenix_project
```

The `750` permission means:
- Owner: read, write, execute
- Group: read and execute
- Everyone else: no access

I also applied the setgid bit to the `src` directory so any file created inside it automatically inherits the group ownership. Without this every developer's files would default to their personal group and break the shared access model.

```bash
sudo chmod 2770 ~/phoenix_project/src
ls -ld ~/phoenix_project/src
```

When `ls -l` shows `drwxrws---` the `s` in the group execute position confirms setgid is active.

**Troubleshooting from this day:**

- Typo in path: typed `phoenix_projects` instead of `phoenix_project`. Linux does not guess. The path has to be exact.
- Got `Operation not permitted` trying to modify a root-owned directory. Fixed by adding `sudo`.
- Learned the difference between running `chmod` on a single directory versus using `-R` to apply changes to the entire tree.

---

## Day 3: User Management and Account Security

The final day was about managing user accounts. Two tasks: create a new user and lock a departing user's account without deleting it.

### Creating a User and Adding to a Group

I created a new account for `b.smith` with a home directory and then added them to the `developers` group.

```bash
sudo useradd -m b.smith
sudo usermod -aG developers b.smith
id b.smith
grep "^developers" /etc/group
```

The `-m` flag tells `useradd` to create the home directory automatically. Without it the account exists but has no home folder.

The `-aG` flag on `usermod` is important. Using `-G` alone would have replaced all of `b.smith`'s existing group memberships with just `developers`. The `-a` flag appends instead of replacing.

### Locking an Account

`j.doe` needed to be removed from active access for a security audit. The right move here was to lock the account rather than delete it.

```bash
sudo passwd -l j.doe
grep "^j.doe" /etc/shadow
```

Locking puts a `!` in front of the password hash in `/etc/shadow` which blocks authentication. The account and all its data stays intact. Deleting the user would wipe the audit trail which is exactly what you need to keep during an investigation.

**Troubleshooting from this day:**

- Mistyped `username` instead of `useradd`. When a command fails with "not found" the first thing to check is whether the command itself is spelled correctly.
- Tried to manage a user that had not been created yet. Got "user does not exist." Running `id b.smith` before modifying anything would have caught this earlier.
- Clarified the difference between `/etc/passwd` and `/etc/shadow`. `/etc/passwd` stores user identity. `/etc/shadow` stores authentication credentials. They are not interchangeable.

---

## Key Lessons

**Verify before acting.** Running `id <username>` or `ls -la` before a modification command takes five seconds and saves a lot of debugging. The workflow that stuck was: check current state, execute the command, verify the state changed correctly.

**The append flag matters.** `usermod -G` is destructive. It replaces a user's entire group list. `usermod -aG` is additive. Before using any command that modifies a list or collection check whether an append variant exists.

**Locking is better than deleting in a compliance context.** Deleted user accounts take audit history with them. A locked account blocks access while keeping everything intact for forensic review.

**Linux is exact.** One extra character in a path, wrong capitalization in a `grep` search or a missing flag can either break a command entirely or give you silently wrong results. Slow down and read the error message before running anything again.

---

## Files in This Repo

| File | Description |
|------|-------------|
| `README.md` | This document |
| `workflow.html` | Visual step-by-step workflow of the full simulation |

---

## Tools Used

- Ubuntu Linux (Bash/Zsh)
- LabEx simulation environment
- Core Linux utilities: `grep` `dmesg` `diff` `chmod` `chown` `useradd` `usermod` `passwd`
