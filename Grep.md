# Grep

**Platform:** TryHackMe
**Category:** Challenge Room
**Difficulty:** Medium
**Status:** ✅ Solved (registered, shell obtained, admin password recovered)

## Background

Grep is a room about searching — fittingly, since almost nothing on the target works the obvious way. The site **SearchMe** (vhost `grep.thm`) will not let a user register: the registration request carries an **API key**, and the key the site itself uses is dead. The way back in is off the box entirely — the application's name, visible in the site header and the TLS certificate, leads to its **public GitHub repository**, and a commit titled like *"fix: remove key"* still contains the real, working API key in the diff. With registration forced through Burp using that key, the first flag falls. The repo also betrays an `upload.php` endpoint that the UI never links to; its file-type filter checks the upload's **magic bytes**, which a hex editor defeats by dressing the PHP shell's header in an image signature. The resulting web shell finds a `users.sql` backup (only the admin's email is needed) and a `leakchecker` tool — feed the admin's email to `check_email.php` and it returns the admin's password from its leak database.

## Lab Objective

Register on SearchMe by recovering the real API key from the application's public GitHub history, log in for the first flag, bypass the upload filter with hex-edited magic bytes to get a web shell, then use the SQL backup and the leak-checker to recover the admin's password.

**Provided resources:** the target machine only; no credentials are given.

<img width="3333" height="944" alt="step-1" src="https://github.com/user-attachments/assets/dc8fde4d-00dd-4f4c-bdbd-0aceaf5ebd8a" />

<img width="3191" height="772" alt="step-2" src="https://github.com/user-attachments/assets/88e669a8-d6b1-4c7b-a3e9-f2e6e7786adf" />

## Walkthrough

### Step 1: First Look — a Default Page on HTTP, a Real Site on HTTPS

Browsing to the target IP over plain HTTP returns only the **Apache2 Ubuntu default page** — a decoy surface. Over **HTTPS**, the picture changes: there is real site information there. The difference is the virtual host, which the scan confirms next.

<img width="2780" height="1019" alt="step-3" src="https://github.com/user-attachments/assets/ae259727-b135-4711-b8e8-6ef22ea4ca80" />

<img width="1543" height="731" alt="step-4" src="https://github.com/user-attachments/assets/e1c9cb28-ef00-4cb2-a4b5-41efdaef4f70" />

### Step 2: Scan the Target

```bash
nmap -Pn -A 10.49.164.42
```

Three ports are open:

| Port | Service | Detail |
|---|---|---|
| 22/tcp | SSH | OpenSSH 8.2p1 Ubuntu |
| 80/tcp | HTTP | Apache 2.4.41 — default page only |
| 443/tcp | HTTPS | Apache 2.4.41 — TLS certificate: **CN `grep.thm`, organisation "SearchME"** |

The certificate names both the vhost and the application. Certificates are public documents — always read them.

<img width="2812" height="1419" alt="step-5" src="https://github.com/user-attachments/assets/70a633fe-5abe-4293-b85c-52e216eb0d30" />

### Step 3: Map the Vhost in `/etc/hosts`

`grep.thm` does not resolve on its own, so add it to Kali's hosts file:

```bash
echo "10.49.164.42 grep.thm" | sudo tee -a /etc/hosts
```

Now the real site loads over HTTPS.

<img width="2044" height="695" alt="step-6" src="https://github.com/user-attachments/assets/192e851d-0f95-4ca1-9fcf-8a527b758947" />

<img width="2730" height="743" alt="step-7" src="https://github.com/user-attachments/assets/4369c173-3b42-477f-80a5-9b20058b4ec9" />

### Step 4: Registration Fails

Try the obvious path first: fill in the registration form and submit. It returns an **error** — accounts cannot be created this way. Something in the request is being rejected; before dissecting it, map the rest of the site.

<img width="2510" height="1566" alt="step-8" src="https://github.com/user-attachments/assets/5af6af66-d78e-44eb-a71a-8974fe3ed77d" />

### Step 5: Enumerate Directories with ffuf

Fuzz the site:

```bash
ffuf -u https://grep.thm/FUZZ -w /usr/share/wordlists/dirb/common.txt
```

Four directories answer. Read the status codes properly: a **403** means the directory *exists* but is forbidden, a **301** means it moved — both are leads, not dead ends. Nothing immediately valuable sits in them, so fuzz *inside* the promising 301 directories as well. Fuzzing `/api` turns up a directory of its own, and following it shows that **`/api/upload` is accessible** — an endpoint the pages never mention.

<img width="2527" height="560" alt="step-9" src="https://github.com/user-attachments/assets/8b863744-b798-4820-a654-5d6181498986" />

<img width="2906" height="342" alt="step-10" src="https://github.com/user-attachments/assets/f9a9aea9-9898-47b1-b730-4618b1d50d3d" />

<img width="2628" height="362" alt="step-11" src="https://github.com/user-attachments/assets/6abcd6b4-d333-4c19-b763-83ef3e595967" />

<img width="1534" height="681" alt="step-12" src="https://github.com/user-attachments/assets/7f1beace-3375-486e-8f38-b60e08b34bdc" />

### Step 6: Read the Registration Request — Find the API Key

Back to the failed registration, this time with the browser's developer tools open. Inspect the **POST request in the Network tab**: the registration call carries an **API key** — but the key the site sends does not work. The mechanism is understood; the value is wrong. A working key has to come from somewhere else.

<img width="2863" height="1418" alt="step-13" src="https://github.com/user-attachments/assets/e01f6291-6259-4343-a26c-b50b7d99af88" />

Repeated fuzzing finds nothing else of value, so change direction: the site header names the application — **SearchMe**, "under development". A named, unfinished app is exactly the kind of thing that lives in public. Enumerate it off-box, through Google and GitHub.

### Step 7: OSINT — the App's GitHub Repository

Search Google for the application; the **first result** is its public GitHub repository. Open it and read the files.

<img width="2822" height="572" alt="step-14" src="https://github.com/user-attachments/assets/336a270f-5bcf-4231-baa5-7432559b0e82" />

<img width="2245" height="841" alt="step-15" src="https://github.com/user-attachments/assets/37899b3c-0f42-409b-8b0e-09df680ae73b" />

<img width="2690" height="1366" alt="step-16" src="https://github.com/user-attachments/assets/5560c934-d800-417c-b1c2-36cfd08f16b5" />

<img width="2936" height="841" alt="step-17" src="https://github.com/user-attachments/assets/4178519a-0dd5-4656-bfec-a55a6708e984" />

### Step 8: The Commit That "Removed" the Key

Inside the repo there is an **api-key folder** — and its history contains a commit along the lines of *"fix: remove key"*. A removal commit is not a removal: the diff shows exactly what was taken out, which is the **real API key** (its value appears in the screenshot and is not repeated here). Submit it as the room's answer — correct. Developers delete secrets from the latest code and forget the history keeps them; this is the same lesson as an exposed `.git`, one platform earlier.

<img width="2746" height="930" alt="step-18" src="https://github.com/user-attachments/assets/22094974-af62-41ce-a4fa-e71528742bf4" />

<img width="2245" height="1577" alt="step-19" src="https://github.com/user-attachments/assets/390b1c92-490a-406b-8bcf-9fd2324a9d43" />

### Step 9: Force Registration Through Burp

Now weaponise the key. Start a normal registration, capture the request in **Burp Suite**, replace the dead API key in the request with the recovered one, and forward it:

<img width="2079" height="1237" alt="step-20" src="https://github.com/user-attachments/assets/bb0ff8f7-7d9b-4879-9b93-14859dc2391d" />

**Registration successful.** The server trusted whichever key the request carried — the browser's broken key was the only thing standing in the way.

<img width="1857" height="990" alt="step-21" src="https://github.com/user-attachments/assets/280c863b-fa13-4dec-b343-ea9682b7db0a" />

### Step 10: Log In — First Flag

Log in with the new account. The dashboard greets the user with the **first flag**.

<img width="2163" height="843" alt="step-22" src="https://github.com/user-attachments/assets/9a84477a-10cb-4e47-8e87-a219eb0e94d4" />

### Step 11: The Hidden Upload Page

The GitHub repo also contained a file named **`upload.php`** that no page links to. Guess its location the simple way: take the logged-in URL, which ends in `dashboard.php`, and swap the filename to `upload.php`. The upload page opens — an authenticated file-upload form, hidden by obscurity alone.

<img width="2119" height="315" alt="step-23" src="https://github.com/user-attachments/assets/c1dcb22c-40f6-4c5f-a038-4755f000331d" />

### Step 12: The Filter — Only Certain File Types

Upload a PHP reverse shell and it is **rejected**. The form's behaviour spells out the rule: only certain **file types** are permitted (the accepted types are listed in the screenshot). The filter is checking what the file *claims to be by its content* — its opening magic bytes — not just its name.

<img width="2238" height="912" alt="step-24" src="https://github.com/user-attachments/assets/4d96a778-9426-490b-96ac-b61595448187" />

<img width="1974" height="679" alt="step-25" src="https://github.com/user-attachments/assets/5c12fd4b-f035-4e3f-8c04-9e32286d022f" />

<img width="2717" height="548" alt="step-26" src="https://github.com/user-attachments/assets/bff9d460-0291-4910-9414-66e52271fe34" />

### Step 13: Hex-Edit the Magic Bytes

So dress the shell as an allowed type. Open the shell file in a hex editor: its first bytes are a placeholder header (`AAAAA...`). Replace those opening bytes with the **hex signature of a permitted image type**, leaving the PHP code below untouched:

<img width="2705" height="445" alt="step-27" src="https://github.com/user-attachments/assets/542a9868-4895-4d85-a8a8-707bd0f0398e" />

<img width="1687" height="463" alt="step-28" src="https://github.com/user-attachments/assets/2e7acb75-c2c1-44c4-8978-716231b8e33f" />

<img width="1834" height="786" alt="step-29" src="https://github.com/user-attachments/assets/a6f57210-6980-4962-8125-aa8b0d9736d5" />

Upload the doctored file — this time the API answers `{"message":"File uploaded successfully."}` The content check passed because the file *starts* like an image; the server will still execute it as PHP because of how it is stored and served.

### Step 14: Execute the Upload, Get the Shell

Browse to the uploads location, where the uploaded file is listed. Click it — the "image" executes, and the listener receives the **web shell**.

<img width="2826" height="954" alt="step-30" src="https://github.com/user-attachments/assets/b358e1b8-60b7-4175-abcd-3a7bde51b167" />

<img width="1342" height="1414" alt="step-31" src="https://github.com/user-attachments/assets/f59be85a-4919-4e0f-b085-3c95b3a3c1a4" />

### Step 15: Explore — the `backup` Folder and `users.sql`

List the working directory and move through what is there. A **`backup`** directory holds an SQL dump, **`users.sql`**:

<img width="1775" height="233" alt="step-32" src="https://github.com/user-attachments/assets/f959d3d0-340a-42d5-b778-eeceff551bbd" />

<img width="1294" height="633" alt="step-33" src="https://github.com/user-attachments/assets/d4361aad-dd42-492e-b841-f3924636ae8e" />

<img width="1035" height="186" alt="step-34" src="https://github.com/user-attachments/assets/99257d62-97b6-436f-a7ab-bc5f0ac3e586" />

<img width="2831" height="896" alt="step-35" src="https://github.com/user-attachments/assets/a4432d71-d3a0-440c-b670-68e73b552452" />

Read it:

```bash
cat users.sql
```

The dump contains the admin's record — hashes, email, everything (values in the screenshot, not repeated here). For the next step, only one field matters: the **admin's email address**.

<img width="791" height="971" alt="step-36" src="https://github.com/user-attachments/assets/626b8154-0df6-4243-8380-5ac67ef433b6" />

### Step 16: The Leak Checker Gives Up the Password

Back in the application directory there is a **`leakchecker`** folder containing **`check_email.php`** — a tool that checks an email address against leaked-password data. Map its subdomain in `/etc/hosts` the same way as the main vhost, open the page, and submit the admin's email recovered from the dump:

<img width="2054" height="938" alt="step-37" src="https://github.com/user-attachments/assets/e24e52e2-4fb6-42e2-9528-d3b7cfd60f5a" />

<img width="1990" height="729" alt="step-38" src="https://github.com/user-attachments/assets/55959209-0805-4c34-83e7-25ab75f8ab2f" />

<img width="1771" height="943" alt="step-39" src="https://github.com/user-attachments/assets/068a7628-23fe-4d6d-b4c4-74a73cc3f269" />

The checker answers with the **admin's password** (shown in the screenshot, not repeated here) — the room's final recovery. A "have I been leaked" feature that returns the actual password is not a checker; it is a credential oracle.

## Key Takeaways

- Committed secrets are never deleted, only hidden by newer commits. The working API key sat in a *"fix: remove key"* diff on a public repo — history is part of the attack surface, on GitHub exactly as in an exposed `.git` folder.
- When a request fails, read the request. The dead API key was visible in the browser's own Network tab; the bug was a value, not a mechanism, and Burp made the value replaceable.
- TLS certificates and site headers are free OSINT: the cert named the vhost (`grep.thm`) and the app (`SearchME`), and the app name led straight to its source.
- File-type filters that trust magic bytes are a costume check. Hex-editing the shell's header to an image signature passed the filter while the code beneath stayed fully executable — real protection validates, converts and stores uploads outside the web root, never merely inspects their first bytes.
- Backups and helper tools finish chains: a `users.sql` in the web tree handed over the admin's identity, and a leak-checker that returns passwords turned an email address into full credentials.

## Remediation

Purge secrets from repository history (rewrite it or rotate the secret — preferably both) and keep production code in private repos; scan commits for keys before pushing. Validate API keys server-side per request and never ship a broken key in client code as the only gate to registration. Authenticate and authorise every endpoint, including unlinked ones like `upload.php` — obscurity is not access control. For uploads, enforce an allow-list by *parsed* content, re-encode images, store files outside the web root with non-executable names, and never let an uploaded file be reachable as code. Keep SQL backups and admin utilities off the web server entirely, and design breach-check features to answer yes/no — never to return the credential itself.
