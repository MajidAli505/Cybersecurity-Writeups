# Remote Code Execution via Web Shell Upload

**Platform:** PortSwigger Web Security Academy
**Category:** File Upload
**Difficulty:** Apprentice
**Status:** ✅ Solved

## Background

A file upload feature becomes remote code execution when three conditions coincide: the application accepts executable file types, stores the upload somewhere web-accessible, and the server *executes* files from that location instead of just serving them. Then an "avatar image" that is actually a PHP script is not stored as data — it is installed as a program, and visiting its URL runs it with the web server's privileges. This lab is the pure form of the bug: the upload performs no validation at all.

## Lab Objective

The site's image upload function stores files on the server without validating them. Upload a basic PHP web shell, use it to read the file `/home/carlos/secret`, and submit the secret via the lab banner.

**Provided resources:** your account `wiener:peter`.

<img width="2202" height="1050" alt="step-1" src="https://github.com/user-attachments/assets/e62251f7-f5e7-45c1-9625-a6a3eaa546d9" />

## Walkthrough

### Step 1: Log In and Find the Upload

Log in and open your account page. It offers an avatar upload — the vulnerable function.

<img width="2700" height="1499" alt="step-2" src="https://github.com/user-attachments/assets/5b2c8e50-d0b9-4d9e-9fe2-032c5399a442" />

<img width="2936" height="1386" alt="step-3" src="https://github.com/user-attachments/assets/b3506200-9fc4-475d-9714-db3eb9201f9c" />

<img width="3329" height="1547" alt="step-4" src="https://github.com/user-attachments/assets/686c91fa-c213-4083-be79-cefc0bb4316f" />

### Step 2: Write a One-Line Web Shell

Create a small PHP file whose entire job is to print the target file's contents:

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

Save it with a `.php` extension. No disguise is needed here — nothing will check what the file claims to be.

<img width="2192" height="794" alt="step-5" src="https://github.com/user-attachments/assets/3b6b8292-4c37-41bb-9d4c-1c6d5dd7f2f3" />

### Step 3: Upload the Shell and Trigger It

Upload the PHP file through the avatar form. It is accepted without complaint. Uploaded avatars are stored under a predictable, web-accessible path, so request the file's URL (the upload request and the file's location are visible in Burp's history): the server executes the PHP instead of downloading it, and the response body is the contents of `/home/carlos/secret` — the secret itself, printed by your shell.

<img width="2627" height="1289" alt="step-6" src="https://github.com/user-attachments/assets/43732a2f-1698-4ca6-bd3a-50398843056a" />

<img width="2603" height="1145" alt="step-7" src="https://github.com/user-attachments/assets/96eff933-5f45-48e7-9f99-546e60230959" />

### Step 4: Submit the Secret

Copy the secret from the response and submit it using the button in the lab banner. The lab is solved.

<img width="2256" height="590" alt="step-8" src="https://github.com/user-attachments/assets/3d002451-0d90-47e6-ae23-83611a24b61b" />

<img width="2707" height="892" alt="step-9" src="https://github.com/user-attachments/assets/77e1554b-f52f-434d-bee4-1e55456d66ff" />

<img width="3086" height="1026" alt="step-10" src="https://github.com/user-attachments/assets/d412debe-9902-49e3-975e-a7db691b2146" />

## Key Takeaways

- Unrestricted upload + web-accessible storage + server-side execution of that directory equals RCE; removing any one of the three breaks the chain.
- The shell needed no cleverness — a single line reading one file — because there was no control to bypass. Severity here is set by the *absence* of defenses, not by payload sophistication.
- Uploaded files must be treated as hostile content from the moment they arrive, even when the feature ("just an avatar") sounds harmless.

## Remediation

Validate uploads on the server against an allowlist of genuinely needed types, verifying the file's actual content rather than its name or declared type; rename stored files to server-generated names; and, most importantly, store uploads outside the web root (or in a location where script execution is disabled) so that even a successful malicious upload can never be executed.
