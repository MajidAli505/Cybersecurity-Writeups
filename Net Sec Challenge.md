# Net Sec Challenge

**Platform:** TryHackMe
**Category:** Learning Room — Network Security (Nmap)
**Difficulty:** Easy
**Status:** ✅ Solved (all questions answered, all flags captured)

## Background

Net Sec Challenge is TryHackMe's hands-on test of **Nmap fundamentals**, built as a set of questions against one live target (10.49.159.122). It is not about the default scan — the room is designed so that a lazy top-1000 scan *misses the point of the exercise*: one service hides on port **10121**, far outside the common range. The room then walks through the rest of the scanning toolkit in order: counting the full port surface with `-p-`, version detection with `-sV` (where two of the "versions" are themselves planted flags), banner grabbing an SSH header, logging in to an FTP service running on a nonstandard port, and interpreting a **NULL scan** (`-sN`), whose `open|filtered` results confuse anyone who has only ever run SYN scans. Every answer below was produced by the scan shown, not guessed.

## Lab Objective

Answer the room's questions against the target: find the hidden high port, count all open TCP ports, extract the planted flags from service banners and an FTP account, identify the nonstandard FTP server's version, solve the mini-challenge on port 8081, and read a NULL scan correctly.

**Provided resources:** the target machine and the room's AttackBox environment.

<img width="3137" height="287" alt="step-1" src="https://github.com/user-attachments/assets/bab5d53c-e83c-4cb1-ab60-62d6c8439758" />


<img width="2226" height="691" alt="step-2" src="https://github.com/user-attachments/assets/a6cde68c-3990-4cad-9f27-be648cb6ef17" />


## Walkthrough

### Step 1: The Hidden Port — Scan Beyond the Top 1000

A default `nmap <target>` only checks the 1000 most common ports. The room states there is an open port **above 10,100** — so scan that range directly:

```bash
nmap -n 10.49.159.122 -p 10100-65535
```

One port answers: **10121/tcp, open**. That is the hidden service — and a reminder that `-p-` (all 65,535 ports) exists precisely because interesting services are often moved off their defaults.

<img width="2161" height="293" alt="step-3" src="https://github.com/user-attachments/assets/9a140c75-e7f7-4a0a-90e1-1dbb91866811" />


<img width="3541" height="700" alt="step-4" src="https://github.com/user-attachments/assets/4838afc0-bb1c-49fa-b60d-7133c803190b" />


### Step 2: Count the Whole Surface — `nmap -p-`

Run the full-range scan to settle the total:

```bash
nmap -p- 10.49.159.122
```

**7 TCP ports are open:**

| Port | Service |
|---|---|
| 22/tcp | SSH |
| 80/tcp | HTTP |
| 139/tcp | NetBIOS-SSN |
| 445/tcp | Microsoft-DS (SMB) |
| 8081/tcp | HTTP (the mini-challenge) |
| 10001/tcp | scp-config |
| 10121/tcp | unknown (identified as FTP below) |
<img width="3804" height="962" alt="step-5" src="https://github.com/user-attachments/assets/6c117571-8b7f-44bf-a162-3f08cfe84a62" />


### Step 3: Version Detection — Port 80's "Version" Is a Flag

Add service/version detection:

```bash
nmap -sV 10.49.159.122
```

On port 80, the VERSION column does not name a real product at all — the server's header has been replaced with a **flag string** (visible in the screenshot, not repeated here), and the `http-server-header` line confirms the same value. Lesson: banners are just text a server chooses to send; read them, because both attackers and challenge authors hide things in them.

<img width="3192" height="419" alt="step-6" src="https://github.com/user-attachments/assets/e95dd25e-ca6d-47d3-b9d6-7737360a9c93" />


### Step 4: Banner Grabbing — the Flag in the SSH Header

The same technique answers the next question: the **SSH server's header** — the banner SSH sends before authentication — contains a planted flag. Version detection (or a raw connection to port 22) displays it; the value is in the screenshot and is not repeated here.

<img width="3543" height="410" alt="step-7" src="https://github.com/user-attachments/assets/324dc40e-8bb7-42a3-a1c8-c26ad360d91a" />


### Step 5: The Nonstandard FTP Server and Its Version

Port 10121's service is **FTP** — the standard service, moved to a nonstandard port. Its banner/version, again from `-sV` or by connecting to it, is **vsftpd 3.0.5**. Moving a service to a high port hides it from default scans, not from anyone who scans the full range and reads banners.

<img width="1662" height="394" alt="step-8" src="https://github.com/user-attachments/assets/f762fc11-06cf-47de-a72b-25d11a187b72" />


### Step 6: The FTP Accounts — `eddie` and `quinn`

The room supplies two usernames learned by social engineering: **eddie** and **quinn**. Log in to the FTP service on 10121 with those accounts and browse their files: one of the two accounts holds a file containing a **flag** (value in the screenshots, not repeated here). This is the practical point of banner and account enumeration — services on odd ports still enforce, and still leak, the same things as services on standard ones.

<img width="2016" height="755" alt="step-9" src="https://github.com/user-attachments/assets/0c71e4c6-c06b-46c1-b03b-c797193d49eb" />


<img width="1992" height="850" alt="step-10" src="https://github.com/user-attachments/assets/d7dd9011-312c-4920-9e98-be3b22668e36" />


<img width="3437" height="760" alt="step-11" src="https://github.com/user-attachments/assets/72de169f-b65e-4dca-8cb3-cf4092d3fb35" />


### Step 7: The Mini-Challenge on Port 8081

Browsing to `http://10.49.159.122:8081` presents a small self-contained challenge page; solving what it asks yields the final web flag (shown in the screenshot, not repeated here).

<img width="3625" height="558" alt="step-13" src="https://github.com/user-attachments/assets/b28ef88b-e54e-41ca-89ad-4c3b9c08cb73" />


### Step 8: Reading a NULL Scan

Finally, the room demonstrates a different scan type. A **NULL scan** sends packets with *no* TCP flags set:

```bash
nmap -sN -p- 10.49.159.122
```

Against this target every one of the 7 ports reports **`open|filtered`** — a NULL scan cannot distinguish the two states here, because a port that sends no reply might be open (silently dropping the probe, per RFC behaviour for open ports) or filtered. The takeaway is to choose the scan type for the question being asked: NULL/FIN/Xmas scans exist to slip past some filters and to fingerprint OS behaviour, not to replace a connect/SYN scan when a definite open/closed answer is needed.

<img width="2048" height="861" alt="step-12" src="https://github.com/user-attachments/assets/d4d17fc5-e886-4cb8-b2fa-405d1e8cc7d1" />


## Key Takeaways

- Default scans lie by omission: the top-1000 list missed port 10121 entirely. When a target matters, scan all 65,535 ports (`-p-`) — or, when the hint gives a range, scan exactly that range and save the time.
- `-sV` output deserves a slow read. Two of this room's answers were planted *in* the version strings and headers themselves; banners are data, not decoration.
- Nonstandard ports are camouflage, not security: FTP on 10121 was still vsftpd, still banner-grabbable, and still holding user files.
- Scan types change the meaning of the output. The same 7 ports are "open" to a connect scan and "open|filtered" to a NULL scan — knowing *why* (no reply is ambiguous) is the difference between reporting a finding and reporting a guess.

## Remediation

Defenders should treat this room as a checklist of what scanning reveals: minimise the listening surface (7 open ports, including legacy NetBIOS/SMB, is a lot), suppress or genericise service banners where the software allows it, and never place flags, internal notes or real credentials in headers and banners. Run services on standard ports behind proper filtering rather than relying on obscurity — a full-range scan is a three-second decision for an attacker. Finally, monitor for the scans themselves: full-range and NULL scans have distinctive signatures that IDS tooling can flag long before the first login attempt.
