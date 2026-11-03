# Web Shell Upload via Extension Blacklist Bypass

**Platform:** PortSwigger Web Security Academy
**Category:** File Upload
**Difficulty:** Practitioner
**Status:** ✅ Solved

## Background

Blacklists fail in a predictable way: they enumerate *bad* extensions, and anything the list forgot remains allowed. PHP alone can hide behind many extensions (`.php5`, `.phtml`, `.phar`, and more), so extension roulette sometimes wins directly. But the deeper miss in this lab is a file that is not code at all: **`.htaccess`**. On Apache servers, a `.htaccess` file in a directory is *configuration* — it can redefine how the server treats other files in that directory. A blacklist focused on script extensions never thinks to block it, yet uploading one lets the attacker rewrite the server's rules: declare that some harmless-looking, invented extension should be handled as PHP, then upload the shell under that extension.

## Lab Objective

The upload function blacklists certain file extensions, but the blacklist's configuration leaves a way through. Upload a basic PHP web shell, read `/home/carlos/secret` with it, and submit the secret via the lab banner.

**Provided resources:** your account `wiener:peter`.

<img width="2160" height="1178" alt="step-1" src="https://github.com/user-attachments/assets/cdd5b7cb-857c-4e40-b88a-23b689a7f6d0" />


## Walkthrough

### Step 1: Probe the Blacklist

Log in and open the avatar upload. The plain `.php` shell is rejected outright — PHP files are not allowed. Try the usual alternate PHP extensions instead: they are blocked too, and the one variant that does slip through is not executed by the server when requested, so it reads as a dead end. The blacklist is doing its job against *code* extensions.

<img width="2694" height="1448" alt="step-2" src="https://github.com/user-attachments/assets/1bc06dbf-7afd-4b0c-9dc2-675f5c5000fe" />


<img width="2672" height="1381" alt="step-3" src="https://github.com/user-attachments/assets/de81d359-03c3-4329-ba70-1909a8a5a8c0" />


<img width="2197" height="865" alt="step-4" src="https://github.com/user-attachments/assets/4011e838-a8f6-4cf0-acb0-71f7cf5ec854" />


<img width="1558" height="454" alt="step-5" src="https://github.com/user-attachments/assets/0ed9253b-cc79-4f0b-be01-f268a039a1e4" />


<img width="2066" height="850" alt="step-6" src="https://github.com/user-attachments/assets/2f75b156-0d55-4918-89e5-a604a8b1b6a9" />


<img width="1853" height="349" alt="step-7" src="https://github.com/user-attachments/assets/d2f608a5-a3b0-43f2-ae9b-2a23ac159e60" />


<img width="1881" height="842" alt="step-8" src="https://github.com/user-attachments/assets/c0ec511a-513a-495d-8c72-22361a31cd88" />


<img width="1988" height="385" alt="step-9" src="https://github.com/user-attachments/assets/ca22a7dc-793c-4296-a3d4-f20d147b7769" />


<img width="2965" height="1144" alt="step-10" src="https://github.com/user-attachments/assets/855ffaa3-d2a1-4fcd-9d58-8ddb9736596b" />


<img width="2861" height="1146" alt="step-11" src="https://github.com/user-attachments/assets/3a946cdd-4971-4154-988e-bc6a396637b1" />


### Step 2: Change the Server's Rules Instead — Upload `.htaccess`

A different angle: the response headers in the proxy history show the site runs on **Apache**, and Apache honors per-directory configuration files. The blacklist never considered configuration files dangerous, so try uploading one. Send the upload request (`POST /my-account/avatar`) to Repeater and reshape it:

- **filename:** `.htaccess`
- **Content-Type:** `text/plain`
- **File contents** (replacing the PHP payload) — a single directive mapping an invented extension to the PHP handler:

```apache
AddType application/x-httpd-php .l33t
```

The upload succeeds: `.htaccess` was not on the blacklist. From now on, Apache will execute any file ending in `.l33t` inside the avatars directory as PHP.

<img width="2653" height="1360" alt="step-12" src="https://github.com/user-attachments/assets/80edc34c-b2f7-4386-85b5-db70f83ef1b1" />

<img width="2701" height="1413" alt="step-13" src="https://github.com/user-attachments/assets/eaf9c427-227c-4f74-9be9-51dbc0b0fe87" />


### Step 3: Upload the Shell Under the Invented Extension

Repeat the upload with the original PHP shell, renaming the file to use the newly blessed extension (in this run, `pann.l33t`). It uploads without complaint — `.l33t` is nobody's blacklist entry.

<img width="2719" height="1181" alt="step-14" src="https://github.com/user-attachments/assets/2e33e5ac-56a7-4401-ae47-11550abb5f0d" />


### Step 4: Execute It

Request the uploaded file:

```http
GET /files/avatars/pann.l33t
```

Thanks to the planted `.htaccess`, Apache routes it to the PHP engine just as if it were a `.php` file. It executes, and the response contains the contents of `/home/carlos/secret`. Submit the secret via the lab banner. The lab is solved.

<img width="2490" height="942" alt="step-15" src="https://github.com/user-attachments/assets/5ebf64d5-5e35-444b-b31a-dafe77dc30e2" />


<img width="3320" height="854" alt="step-16" src="https://github.com/user-attachments/assets/3ed1d774-9534-41c3-adfc-3992b9b44263" />


<img width="3054" height="920" alt="step-17" src="https://github.com/user-attachments/assets/7521f8dc-ae10-41f8-9516-1e211856244c" />


## Key Takeaways

- A blacklist defends a list, not a property. It blocked every extension that *was* PHP and missed the file that could *make* something PHP.
- Configuration files are code-adjacent: `.htaccess`, `web.config`, and similar artifacts change how the server behaves and must be treated as at least as dangerous as scripts.
- Server fingerprinting mattered: spotting Apache in the response headers is what turned "all extensions blocked" into "rewrite the server's extension mapping."

## Remediation

Use an extension/content **allowlist** (only the image types the feature needs) instead of a blacklist, validate file contents server-side, and explicitly reject configuration filenames such as `.htaccess` and `web.config`. Store uploads outside the web root or with execution disabled, disable `.htaccess` overrides for upload directories (`AllowOverride None`), and serve user files from a separate, hardened context.
