# Information Disclosure in Version Control History

**Platform:** PortSwigger Web Security Academy  
**Category:** Information Disclosure  
**Difficulty:** Practitioner  
**Status:** ✅ Solved

## Overview

Deploying a web application straight from a working copy is a common shortcut — and it drags the entire `.git` directory onto the web server with it. If the server doesn't block access to that directory, anyone can download the repository metadata and reconstruct not just the current source code, but the **complete history**: every commit, every deleted file, and every secret that was ever committed and later "removed". Deleting a password from a file does not delete it from history — the old version remains one `git show` away.

## Lab Objective

Recover the `administrator` password from the site's exposed version control history, log in as administrator, and delete the user `carlos`.

<img width="2202" height="742" alt="step-1" src="https://github.com/user-attachments/assets/e3ca9e76-cff5-4f25-a78c-ca826e7d062e" />

## Walkthrough

### Step 1: Browse to /.git

Open the lab and navigate to `/.git`. The server returns the lab's Git version control data — the repository was deployed with its metadata exposed.

<img width="2018" height="1141" alt="step-2" src="https://github.com/user-attachments/assets/6df6993b-458a-4a58-ac74-f3e4909efe41" />

### Step 2: Download the Repository

Download a full copy of the exposed directory. On Linux, a recursive `wget` does the job:

```bash
wget -r https://YOUR-LAB-ID.web-security-academy.net/.git/
```

Once downloaded, open the folder and confirm it contains a working `.git` repository that Git can read locally.

<img width="2331" height="375" alt="step-3" src="https://github.com/user-attachments/assets/4b86a008-f1d4-49cc-a44b-64b2ec7d7408" />

### Step 3: Review the Commit History

Inside the downloaded repository, list the commits:

```bash
git log
```

The history includes a suspicious latest commit whose message says the admin password was removed from the config — note its commit ID (hash).

<img width="2389" height="744" alt="step-4" src="https://github.com/user-attachments/assets/b207cc1c-f062-4410-8097-7bcca85fe72d" />

### Step 4: Inspect the "Removal" Commit

Show that commit's changes:

```bash
git show <commit-id>
```

The diff for `admin.conf` tells the whole story in two lines: the old line with the plaintext `ADMIN_PASSWORD` value was deleted, and replaced with a line that reads the password from an environment variable instead. The fix was correct going forward — but the diff itself preserves the original password forever. Note it down.

<img width="2563" height="875" alt="step-5" src="https://github.com/user-attachments/assets/585e6976-a257-4daa-84b4-4dbf19a5f63a" />

### Step 5: Log In as Administrator

Back in the lab, log in with the username `administrator` and the password recovered from the commit history.

<img width="2609" height="1401" alt="step-6" src="https://github.com/user-attachments/assets/c250e876-d539-4d89-9066-65dad45a56ab" />
<img width="3021" height="1316" alt="step-7" src="https://github.com/user-attachments/assets/a090c4c4-bd8a-4349-ad6e-3264bffae27b" />

### Step 6: Delete Carlos

With administrator access, open the admin panel and delete the user `carlos`.

<img width="3040" height="840" alt="step-8" src="https://github.com/user-attachments/assets/e7a343a8-451f-4157-a939-9d4a03da2bb6" />

### Step 7: Lab Solved

The panel confirms the user was deleted successfully, and the lab is complete.

<img width="3116" height="1039" alt="step-9" src="https://github.com/user-attachments/assets/abe7c4f3-72a7-488b-9da3-fea17bcb814e" />

## Key Takeaways

- An exposed `.git` directory discloses the entire project: source code, configuration, and full history, including things that were later deleted. Tools and a single recursive download are all it takes.
- "Removed from the file" is not "removed from the repository". A secret that was ever committed must be treated as compromised: rotate it, and rewrite history if the repository is shared.
- Commit messages can be signposts — a message about removing a password tells an attacker exactly which commit to inspect.

## Remediation

- Never deploy from a working copy with version control metadata included; exclude `.git` (and equivalents such as `.svn` and `.hg`) from deployments, and block those paths at the web server as defense in depth.
- Keep secrets out of repositories entirely — use environment variables or a secrets manager from the start, not as a later cleanup.
- If a secret does reach a repository, rotate it immediately and purge it from history (for example with `git filter-repo`), assuming anyone could already have a copy.
