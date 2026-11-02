# Source Code Disclosure via Backup Files

**Platform:** PortSwigger Web Security Academy  
**Category:** Information Disclosure  
**Difficulty:** Apprentice  
**Status:** ✅ Solved

## Overview

Backup copies of source files are a classic leftover: an editor saves `file.java.bak`, an old version is parked in a `/backup` folder "just in case", and the web server happily serves it to anyone who asks — as plain text, because a `.bak` extension is not executed as code. The result is full source code disclosure, and source code tends to contain the crown jewels: hard-coded credentials, connection strings, internal logic, and comments that map out further attacks.

## Lab Objective

Locate the backup file that leaks this application's source code, extract the database password that is hard-coded in it, and submit that password as the solution.

<img width="2171" height="777" alt="step-1" src="https://github.com/user-attachments/assets/788098bf-1663-4f74-93a0-e7b615c96e77" />

## Walkthrough

### Step 1: Check robots.txt

A good first stop in any recon is `/robots.txt`, which lists paths the site asks crawlers not to visit. Here it discloses the existence of a `/backup` directory — not linked anywhere in the application, but now known.

<img width="1969" height="353" alt="step-2" src="https://github.com/user-attachments/assets/72931579-a148-48a8-9ac5-39175644d305" />

### Step 2: Browse the Backup Directory

Visit `/backup`. The directory contains a backup source file:

```text
ProductTemplate.java.bak
```

<img width="1830" height="483" alt="step-3" src="https://github.com/user-attachments/assets/47a06541-1296-4501-8d66-0dbe72d08caf" />

### Step 3: Read the Leaked Source Code

Browse to the file itself:

```text
/backup/ProductTemplate.java.bak
```

The browser displays the complete Java source of the `ProductTemplate` class. In the `readObject` method, a database connection is built with every connection detail hard-coded as a string argument — the PostgreSQL driver, host (`localhost`, port 5432), database and user names, and, as the final argument, the database password in plaintext.

<img width="2922" height="1902" alt="step-4" src="https://github.com/user-attachments/assets/c22741bc-9d42-4eea-b5a2-f6745781ef59" />

### Step 4: Submit the Database Password

Return to the lab, click **Submit solution**, and enter the password taken from the connection builder in the leaked source.

<img width="3207" height="1154" alt="step-5" src="https://github.com/user-attachments/assets/4b35c718-70e3-4cd8-bc1c-0fbeae5fb786" />

### Step 5: Lab Solved

The submission is accepted. One forgotten backup file was enough to hand an unauthenticated visitor both the application's source code and its production database credential.

<img width="3267" height="1092" alt="step-6" src="https://github.com/user-attachments/assets/441f1e12-785a-49d0-83ac-68af17c5d6ad" />

## Key Takeaways

- Anything placed under the web root is public, whatever its extension. Backup suffixes such as `.bak`, `.old`, `.save`, or `~` turn source files into downloadable text.
- `robots.txt` is a recon gift: paths listed there to keep crawlers out are a ready-made list of interesting locations to check manually.
- Hard-coded credentials in source code become critical the moment the source leaks. Secrets belong in environment configuration or a secrets manager, never in code — and they must be rotated if exposure is suspected.

## Remediation

- Never store backups, archives, or old versions of files inside the web root; keep them outside the document root or remove them entirely.
- Configure the server to deny requests for backup and temporary file extensions.
- Remove credentials from source code, load them from secure configuration, and rotate any secret that may have been exposed.
- Treat `robots.txt` as a public document — never list sensitive paths in it.
