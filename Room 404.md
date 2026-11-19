# Room 404

**Platform:** TryHackMe
**Category:** Challenge Room
**Difficulty:** Easy
**Status:** ✅ Solved (flag captured)

## Background

Room 404 is a hotel website where nothing works — every link and even the "Reserve a stay" button leads nowhere useful, which is the room's joke and its hint: stop using the site and start scanning the server. Nmap's scripts report the real finding, an **exposed `.git` directory** in the web root. A plain `git clone` cannot take it (an HTTP-served `.git` folder is not a cloneable remote), but the files are publicly readable, so **git-dumper** fetches the objects one by one and reconstructs the repository locally. The recovered history holds exactly **one commit** — and inspecting that single commit (`git show`) hands over the flag. It is the same lesson as Grep's GitHub history and Borderland's dumped repo, from the third angle: version-control metadata left on a web server is a full source disclosure.

## Lab Objective

Discover the exposed `.git` directory on the hotel's web server, reconstruct the repository with git-dumper, and extract the flag from its commit.

**Provided resources:** the target machine only; no credentials are given.

![step-1](https://github.com/user-attachments/assets/fa641fcf-9117-49d5-b7d8-916e4796cc0a)

<img width="2994" height="735" alt="step-2" src="https://github.com/user-attachments/assets/4b0ea08d-04e4-40fc-b553-c20e6c393937" />

<img width="2756" height="1450" alt="step-3" src="https://github.com/user-attachments/assets/cc7d5aeb-3ff8-46d9-8461-26d5ae93c0c2" />

## Walkthrough

### Step 1: Browse the Site — Everything Is Broken

Open the website. It presents a hotel with links — but the links **do not work**. Even the **"Reserve a stay"** button just leads to another dead page. A site this broken is telling you the intended surface is not the site itself.

<img width="2145" height="976" alt="step-4" src="https://github.com/user-attachments/assets/84c9c8fb-4da5-43a0-a0e1-86e5ed354a46" />

<img width="2858" height="1374" alt="step-5" src="https://github.com/user-attachments/assets/df016bd6-fbcc-46d9-a0a9-42099714624b" />

### Step 2: Scan — Nmap Reports a Git Repository

Run Nmap with its default scripts:

```bash
nmap -sC -sV 10.49.x.x
```

Alongside the basic service information, the scan reports a **git repository** exposed on the web server — the `http-git` style finding that a `.git` directory is sitting in the web root, publicly reachable.

<img width="1540" height="339" alt="step-6" src="https://github.com/user-attachments/assets/8e592ac3-e364-4222-a5db-d18e830b8cc3" />

### Step 3: Confirm the Exposed `.git`

Browse to the `.git` path on the site. The server answers with repository information rather than a 403 — the directory really is being served.

<img width="2768" height="1405" alt="step-7" src="https://github.com/user-attachments/assets/fbb27818-11d5-478d-8d7d-e572e340ef42" />

### Step 4: `git clone` Fails — Check What Is Actually Accessible

The first instinct is to clone it. Create a directory on Kali and try — with two different command forms — and both **fail**: a `.git` folder served as static files over HTTP is not a git remote, so the clone protocol has nothing to talk to.

Before reaching for heavier tools, verify what *is* true: request a known file from inside the `.git` directory directly (for example its `HEAD`) with curl. The output confirms the repository files are **publicly accessible** one by one — which is all a dumper needs.

<img width="1609" height="994" alt="step-8" src="https://github.com/user-attachments/assets/a5efb364-4f85-4a13-bbf2-846d26118865" />

<img width="2205" height="1214" alt="step-9" src="https://github.com/user-attachments/assets/0e6bcabd-9f02-42ac-9019-e2f9a589a467" />

<img width="2254" height="272" alt="step-10" src="https://github.com/user-attachments/assets/f5da8def-be66-497f-a5d9-bc39ebaf5e78" />

### Step 5: Install git-dumper and Reconstruct the Repo

Install **git-dumper** on Kali — the tool built for exactly this case — and point it at the exposed directory. It walks the `.git` structure over HTTP, downloads the objects, and **reconstructs the full repository** locally:

```bash
git-dumper http://10.49.x.x/.git/ ./room404-repo
```

<img width="2078" height="575" alt="step-11" src="https://github.com/user-attachments/assets/8b960dc8-f887-4b66-842a-197cf11e0332" />

<img width="2655" height="685" alt="step-12" src="https://github.com/user-attachments/assets/8b8f718a-a13c-41d7-b701-00842cd662ba" />

<img width="2473" height="231" alt="step-13" src="https://github.com/user-attachments/assets/03ef3e84-8535-4225-86f8-19587b03dffd" />

<img width="2069" height="1443" alt="step-14" src="https://github.com/user-attachments/assets/8a143bc6-f299-4013-827f-6b4b259425bf" />

### Step 6: Read the History — One Commit

Enter the reconstructed repository, list its contents including hidden files, and read the history in short form across all refs:

```bash
cd room404-repo
ls -la
git log --oneline --all
```

The history is minimal: a **single commit**, `0f13550`. One commit means one thing to inspect.

<img width="2339" height="1155" alt="step-15" src="https://github.com/user-attachments/assets/b4aef760-4c24-482b-9764-4b9c2122b8dd" />

<img width="1980" height="763" alt="step-16" src="https://github.com/user-attachments/assets/9d078a21-742f-480c-9107-cd52751889d2" />

### Step 7: Inspect the Commit — Flag

Show that commit in full:

```bash
git show 0f13550
```

Its contents include the **flag** (value in the screenshot, not repeated here). Room complete — the site's buttons never worked because the answer was never in the site; it was in the repository the site was deployed from.

<img width="2424" height="1106" alt="step-17" src="https://github.com/user-attachments/assets/d5e20a28-4ae5-4511-9dc0-6449f286649c" />

## Key Takeaways

- When the application is a dead end, scan the server: Nmap's HTTP scripts flag exposed `.git` directories that no amount of clicking would have found.
- `git clone` failing does not mean the repo is safe or unreachable — a statically served `.git` cannot speak the clone protocol, but every object in it is an ordinary HTTP download. git-dumper exists precisely for that gap.
- Verify before tooling up: one curl to `.git/HEAD` proved public accessibility and justified the whole approach.
- A single-commit history is still a disclosure. Whatever was in the tree when it was committed — source, config, flags, secrets — is recoverable by anyone, forever, unless the directory is removed from the web root.

## Remediation

Never deploy a site by serving its working repository: export a clean build, and block `.git` (and all dotfile/VCS paths) at the web server with an explicit deny rule, verified from outside. Treat any repository that has ever been web-accessible as fully disclosed — rotate every secret it contained and scrub or rebuild its history. Add exposed-VCS checks to routine external scanning; this is one of the most common and most preventable source-code leaks.
