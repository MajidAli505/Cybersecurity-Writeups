# Tomghost

**Platform:** TryHackMe
**Category:** Challenge Room
**Difficulty:** Easy
**Status:** ✅ Solved (user flag and root flag captured)

## Background

Tomghost is built around **Ghostcat (CVE-2020-1938)**, a file-read / file-inclusion flaw in Apache Tomcat's AJP connector. Tomcat normally expects its AJP port (8009) to be reached only by a trusted front-end web server, but when the port is exposed directly, an unauthenticated attacker can speak AJP to Tomcat and read arbitrary files from the web application — including configuration files that were never meant to be public, such as `/WEB-INF/web.xml`. In this room that single file read hands over working credentials, and the rest of the chain is built from what those credentials unlock: an SSH login, a PGP-encrypted credential file whose private key is protected by a crackable passphrase, a second user account, and finally a sudo rule that a GTFOBins entry turns into a root shell.

## Lab Objective

Exploit the exposed AJP service on the target to recover credentials, work through the PGP-protected credential file to reach a second user, and escalate to root — capturing the user flag and the root flag on the way.

**Provided resources:** the target machine only; no credentials are given.

<img width="2937" height="630" alt="step-1" src="https://github.com/user-attachments/assets/b5d874e7-6522-45e8-ad01-27df0596a933" />


<img width="3269" height="1564" alt="step-2" src="https://github.com/user-attachments/assets/5ecc6438-3f07-4b9f-8a78-ee329e9fd7b7" />


## Walkthrough

### Step 1: Scan the Target

Run an aggressive Nmap scan against the target:

```bash
nmap -A -Pn 10.48.158.101
```

Four ports are open:

| Port | Service | Detail |
|---|---|---|
| 22/tcp | SSH | OpenSSH 7.2p2 Ubuntu |
| 53/tcp | tcpwrapped | — |
| 8009/tcp | AJP13 | Apache Jserv (Protocol v1.3) |
| 8080/tcp | HTTP | Apache Tomcat 9.0.30 |

Tomcat 9.0.30 with its AJP connector exposed on 8009 is the Ghostcat pattern — that is the way in.

<img width="2330" height="1040" alt="step-3" src="https://github.com/user-attachments/assets/c5b24aa6-2981-40fa-9deb-591d3337fbaa" />


### Step 2: Read `web.xml` Through Ghostcat with Metasploit

Metasploit ships an auxiliary module for exactly this vulnerability. Load it, point it at the target, and keep the default file — `/WEB-INF/web.xml` is where web applications keep their configuration:

```text
msfconsole -q
use auxiliary/admin/http/tomcat_ghostcat
set RHOSTS 10.48.158.101
run
```

The module's defaults are already aimed correctly: `RPORT 8009` and `FILENAME /WEB-INF/web.xml`.

<img width="2512" height="1068" alt="step-4" src="https://github.com/user-attachments/assets/2f72062a-655f-441c-b7d2-e53912f09bfd" />


<img width="2104" height="766" alt="step-5" src="https://github.com/user-attachments/assets/e84f9bea-6666-4850-a094-a011a436b64f" />


### Step 3: Harvest the Credentials and Log In over SSH

Reading the module's output carefully pays off. The returned `web.xml` describes the "Welcome to GhostCat" application — and its description element contains a plaintext username and password pair for the user **skyfuck** (the password itself is visible in the screenshot below and is not repeated here).

Those credentials are not just for the web application. Try them against SSH:

```bash
ssh skyfuck@10.48.158.101
```

The login succeeds — a shell on the target as `skyfuck`. Credentials stored in a readable config file rarely protect only one service; password reuse turns a web file-read into host access.

<img width="2643" height="658" alt="step-6" src="https://github.com/user-attachments/assets/94ebfa04-eb7b-4cf7-8294-e31a2fe74580" />

<img width="2333" height="1456" alt="step-7" src="https://github.com/user-attachments/assets/ed60d64a-5b3a-4291-827a-b6fcfd0b9e05" />


### Step 4: Find the PGP Files

Listing the home directory shows two files that clearly belong together:

- `credential.pgp` — a PGP-encrypted file, presumably holding a secret.
- `tryhackme.asc` — an ASCII-armoured PGP key file.

An encrypted message and a key, left side by side: the obvious plan is to import the key and decrypt the message.

<img width="2116" height="203" alt="step-8" src="https://github.com/user-attachments/assets/155daddd-598a-45c3-ac91-cb882d1f0b14" />


### Step 5: Import the Key — the Decryption Is Passphrase-Locked

Import the key into the local GPG keyring:

```bash
gpg --import tryhackme.asc
```

The key imports — it is the private key needed for the message. But attempting to decrypt `credential.pgp` stops at a prompt: the private key itself is protected by a **passphrase**, and guessing it is not going to work. The key has to be attacked offline instead.
<img width="2591" height="883" alt="step-9" src="https://github.com/user-attachments/assets/7d3e8681-a903-418b-b33c-7f19321c9f88" />

<img width="2165" height="793" alt="step-10" src="https://github.com/user-attachments/assets/41cea07d-01a3-48d5-b8d2-756c9dd5c7cf" />


### Step 6: Copy the Files to Kali and Crack the Key's Passphrase

Copy both files down to the Kali machine, where the cracking tools live:

```bash
scp skyfuck@10.48.158.101:~/credential.pgp skyfuck@10.48.158.101:~/tryhackme.asc .
```

Convert the imported key into a format John the Ripper understands with `gpg2john`, which parses PGP private keys into a crackable hash:

```bash
gpg2john tryhackme.asc > pgp_hash
```

<img width="2818" height="1240" alt="step-11" src="https://github.com/user-attachments/assets/df78c4ab-5e8d-4ebd-8299-09a8005e6449" />


<img width="1912" height="566" alt="step-12" src="https://github.com/user-attachments/assets/976353ab-d511-43dd-a947-900cffdcd030" />


Run John against the hash with the standard wordlist:

```bash
john pgp_hash --wordlist=/usr/share/wordlists/rockyou.txt
```

The passphrase falls quickly — it is a weak, dictionary word (shown in the screenshot, not repeated here). A strong encryption scheme was undone by the human-chosen secret protecting it.

<img width="2845" height="1071" alt="step-13" src="https://github.com/user-attachments/assets/92bcae3b-b44d-433c-ba47-befa35477dda" />


### Step 7: Decrypt the Credentials, Switch User, Take the User Flag

Back on the target, decrypt the credential file and enter the cracked passphrase when prompted:

```bash
gpg --decrypt credential.pgp
```

The plaintext contains the password of a second user, **merlin**. Switch to that account:

```bash
su merlin
```

<img width="2709" height="946" alt="step-14" src="https://github.com/user-attachments/assets/310ebd66-3343-46f8-8e00-0f0271ef4ff0" />

<img width="1644" height="390" alt="step-15" src="https://github.com/user-attachments/assets/c05af1f6-ff72-456c-a68b-4c7ebc7201f0" />


As `merlin`, the first flag — the user flag — can be read from the account's files.

<img width="2327" height="845" alt="step-16" src="https://github.com/user-attachments/assets/f23cd706-8f26-4990-97ad-de3fd968ac32" />


### Step 8: Escalate to Root Through `zip` and GTFOBins

Check what `merlin` may run with sudo:

```bash
sudo -l
```

One entry stands out: `merlin` can run **zip** as root without a password. Before inventing anything, check GTFOBins — the curated list of Unix binaries that can escape to a shell when run with elevated privileges. Searching it for `zip` shows a **Shell** function for exactly this situation.

<img width="2603" height="460" alt="step-17" src="https://github.com/user-attachments/assets/65de772c-2f80-4f52-8bf0-8a6c4f6a6af6" />


<img width="2322" height="862" alt="step-18" src="https://github.com/user-attachments/assets/bdda4472-7f87-4ca7-87d7-6289f41a4d85" />


<img width="2362" height="1121" alt="step-19" src="https://github.com/user-attachments/assets/032d99af-ee00-4ab4-bfb7-c6a01e958988" />


The GTFOBins technique abuses zip's test-mode command execution: zip is told to "test" an archive by running a command, and that command is a shell — which inherits root from sudo. Following the entry, with `bash` substituted for `sh` as the shell:

```bash
TF=$(mktemp -u)
sudo zip $TF /etc/hosts -T -TT 'bash #'
```

A root shell opens.

<img width="2275" height="256" alt="step-20" src="https://github.com/user-attachments/assets/b8456686-9ef8-41e0-9983-9ee48842dd6d" />


With root, read the root flag from `/root` to finish the room.

<img width="1962" height="295" alt="step-21" src="https://github.com/user-attachments/assets/85e1c1b7-fe56-4ba9-a5f6-77355e57c1b8" />


## Key Takeaways

- An exposed AJP port is not a minor finding: Ghostcat turns it into unauthenticated file read inside the web application, and config files like `web.xml` regularly contain live credentials. AJP should only ever listen on trusted interfaces.
- Read exploit output fully. The credentials were not in a users table or a password file — they were sitting in an XML *description* element, easy to scroll past.
- The PGP chain shows layered failure: the encrypted file and its key were stored together, and the key's passphrase was dictionary-weak, so `gpg2john` + John reduced "encrypted credentials" to a short offline attack. Encryption is only as strong as the passphrase and key storage around it.
- `sudo -l` plus GTFOBins is a complete privilege-escalation method: any binary granted passwordless sudo should be checked against GTFOBins before deeper enumeration, because entries like `zip` convert directly into root shells.

## Remediation

Bind Tomcat's AJP connector to localhost or disable it when no front-end server uses it, require a shared secret for AJP where it must stay enabled, and keep Tomcat patched beyond the Ghostcat fix. Never store real credentials in `web.xml` descriptions or comments, and never reuse application credentials for system accounts such as SSH. Private keys must not be stored next to the data they protect, and key passphrases should be long and non-dictionary — ideally the key material lives in a dedicated secrets store instead of a user home directory. Finally, grant sudo rights narrowly: a passwordless sudo rule for a general-purpose archiving tool is effectively a root grant, as the GTFOBins entry demonstrates.
