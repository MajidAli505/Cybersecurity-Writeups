# Borderland

**Platform:** TryHackMe
**Category:** Challenge Room
**Difficulty:** Medium
**Status:** ✅ Solved (all keys recovered)

## Background

Borderland is not a shell-and-escalate room — it is a hunt for keys hidden across everything a careless deployment leaves behind. The target runs a PHP site for "Context Information Security" (a HackBack 2 scenario), and the answers are scattered over three different exposure mistakes. First, the site offers a downloadable **Android APK**, and mobile apps are just archives: decompiling it surfaces an encrypted key string that a classical cipher (Vigenère, keyed with the company name) barely protects. Second, the web server exposes the site's **`.git` repository**, so the entire PHP source can be reconstructed with git-dumper and read offline — including keys hardcoded in `api.php` and `home.php`, one of which is a decoy that no longer works. Third, the *current* source is not the whole story: the repository's **history** still holds an earlier commit whose keys differ from today's. Each layer rewards the same habit — when a system hands you its artefacts, read all of them, including the old versions.

## Lab Objective

Recover the room's keys from the three exposures: the encrypted key inside the downloadable APK, the working key in the site's dumped source code, and the key preserved in the repository's git history.

**Provided resources:** the target machine only; no credentials are given.

<img width="2945" height="617" alt="step-1" src="https://github.com/user-attachments/assets/08a845bb-e128-4ee2-97d2-6746bd405b7a" />


<img width="3219" height="1609" alt="step-2" src="https://github.com/user-attachments/assets/62b5bfe4-ead0-45f7-9fd8-e0b0f8325790" />


## Walkthrough

### Step 1: Scan the Target

Run a full service scan:

```bash
nmap -Pn -T4 -sVC -A 10.48.172.44
```

Only two ports are open:

| Port | Service | Detail |
|---|---|---|
| 22/tcp | SSH | OpenSSH 7.2p2 Ubuntu |
| 80/tcp | HTTP | nginx 1.14.0 — site title "Context Information Security - HackBack 2", PHP session cookies |

A PHP application behind nginx is the whole surface. SSH has no credentials yet, so the website is where the work happens.

<img width="1798" height="1283" alt="step-3" src="https://github.com/user-attachments/assets/e5138fc7-6c81-406f-b03f-3c654c29e951" />


### Step 2: Browse the Site and Find the APK Download

Open the site in the browser and read the landing page. Among its content is a download link — clicking it delivers an **APK file**, the organisation's Android app, served straight from the web server.

<img width="2194" height="1220" alt="step-4" src="https://github.com/user-attachments/assets/7f177a81-6664-421a-8312-cd77daef77bc" />


<img width="2336" height="746" alt="step-5" src="https://github.com/user-attachments/assets/411f089b-90dc-4d99-b510-230484c8f19e" />


### Step 3: Decompile the APK and Find the Encrypted Key

An APK is a zip of compiled Android code; the tool for reading it comfortably is **JADX** (GUI version), which decompiles it back toward Java source. Download the APK, install JADX on Kali, and load the file.

With the code open, search the sources for the word **`encrypted`**. The search lands on a hardcoded, encrypted key string inside the app — the first secret, protected only by whatever cipher the developer chose.
<img width="1965" height="1195" alt="step-6" src="https://github.com/user-attachments/assets/3e3fc8b1-be35-468b-bc7f-637feb5eb3bf" />


### Step 4: Break the Cipher in CyberChef

Take the encrypted string to CyberChef. The protection turns out to be a **Vigenère cipher**, and the key is the most guessable word in the scenario — `CONTEXT`, the company's own name. Running **Vigenère Decode** with that key returns the plaintext key (the values themselves are visible in the screenshot and are not repeated here). First key recovered — "encryption" with a classical cipher and a company-name key is obfuscation, not security.

<img width="2453" height="1218" alt="step-7" src="https://github.com/user-attachments/assets/577e0987-d5e1-46b1-82ff-ce14edd6b578" />


### Step 5: Dump the Exposed `.git` Repository

The next key needs the site's source code. The server exposes its **`.git`** directory, but pointing a browser at it does not hand over a usable repo — the files come back piecemeal or as errors. That is exactly the case **git-dumper** exists for: it fetches the `.git` objects one by one over HTTP and reconstructs the full repository locally. Install it and dump the repo to Kali:

```bash
git-dumper http://10.48.172.44/.git/ ./borderland-git
```

<img width="2019" height="984" alt="step-8" src="https://github.com/user-attachments/assets/639f9a38-a73e-45b6-aeb5-ffc58377f7ed" />


Work through the recovered source tree — checking the files, their contents, and what the application actually ships to production.

<img width="2516" height="701" alt="step-9" src="https://github.com/user-attachments/assets/84e71fa8-756c-4588-88d9-60aec9388812" />


### Step 6: Read the PHP Source — One Decoy Key, One Working Key

Display the API file:

```bash
cat api.php
```

It contains **two keys** — promising, except that neither of them works. They are stale or decoy values; finding *a* key is not the same as finding *the* key, and every candidate has to be tested before it is trusted.

<img width="2840" height="747" alt="step-10" src="https://github.com/user-attachments/assets/78afce5f-3dd9-4363-9f2d-276bced52264" />


Read the main page's source next:

```bash
cat home.php
```

Here a different key is hardcoded — and this one **works**. Second key recovered, from source code that should never have been downloadable in the first place.

<img width="2643" height="907" alt="step-11" src="https://github.com/user-attachments/assets/ae4966f7-9547-4ec4-9a81-998082d071e0" />


### Step 7: Mine the Git History

The working tree is only the latest state of the code. For the final key, look at what the repository remembers:

```bash
git log
```

The history lists earlier commits. A previous version of the code — from before the keys were rotated — is still fully preserved in the repo, and displaying that commit's content shows its keys.

<img width="1949" height="1387" alt="step-12" src="https://github.com/user-attachments/assets/411d7285-b311-4ff9-a8b8-c3bc6013966c" />

<img width="2685" height="1352" alt="step-13" src="https://github.com/user-attachments/assets/6363aa41-ebef-4546-b89a-5d0ec32fc1b8" />


The historical commit yields the final set of keys (visible in the screenshot, not repeated here). Deleting a secret from the current file never deletes it from git — the history *is* part of the exposure.

<img width="2847" height="1360" alt="step-14" src="https://github.com/user-attachments/assets/f25d92a4-796b-46aa-8ed7-9b7743a9c105" />


## Key Takeaways

- Anything shipped to a client is public. An APK decompiles in minutes with JADX, and a keyword search for "encrypted" or "key" goes straight to the hardcoded secrets; classical ciphers like Vigenère, especially keyed with the company name, add no real protection.
- An exposed `.git` directory is a full source-code disclosure. When the browser cannot fetch it usefully, git-dumper reconstructs the repository object by object — defenders should assume attackers always can.
- Test every key you find. The source contained plausible-looking keys that did not work; the working one sat in a different file. Artefacts lie by staleness as often as by design.
- Git history is forever. Rotating a key in a new commit while the old commit stays in a publicly downloadable repo is not rotation — the old secret must be treated as compromised and the history scrubbed or the repo kept off the web root entirely.

## Remediation

Never embed secrets in mobile apps or client-side code — keys that must stay secret belong on a server, behind authenticated APIs. Use real cryptography ( platform keystores, proper AEAD ciphers) for anything stored on-device, and assume any client binary will be decompiled. Block access to `.git` and other VCS metadata at the web server, and deploy sites by exporting a clean build rather than serving the working repository. When a secret has ever been committed, rotate it everywhere it was used *and* purge it from history (or rebuild the repository), because the commit log is part of the attack surface. Finally, avoid hardcoding keys in PHP source at all — load them from environment or a secrets manager so source disclosure is not credential disclosure.
