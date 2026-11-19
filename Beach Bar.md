# Beach Bar

**Platform:** TryHackMe
**Category:** Challenge Room
**Difficulty:** Medium
**Status:** ✅ Solved (shell obtained, service password recovered)

## Background

Beach Bar runs a jukebox-style web app, and it falls to two classics in sequence. First, the login page's own **HTML comments** contain working credentials (`dj` / `dj`) — source code is served to every visitor, comments included. Inside, the app can **export its data as a YAML file**, and the stack that parses YAML on the way back in is Python's **PyYAML used unsafely**: a crafted YAML document with a `python/object/apply` tag makes the deserialiser *execute a system command* while "loading" it — unauthenticated-format RCE through a data file. That yields a shell and the user flag. The second lesson waits in the source code: the jukebox service is launched with its **password as a command-line argument**, and command lines are world-readable in the process list — `ps` prints the service's password to any local user.

## Lab Objective

Log in with the credentials leaked in the page source, confirm and exploit the YAML deserialization vulnerability for a shell and the user flag, then recover the jukebox service's password from the process list.

**Provided resources:** the target machine only; the login credentials are discoverable in the page source.

<img width="3239" height="729" alt="step-1" src="https://github.com/user-attachments/assets/b76e51de-6501-41f0-a255-10434a349f4b" />

<img width="2358" height="1301" alt="step-2" src="https://github.com/user-attachments/assets/261d2698-4093-485d-acbc-ea91735c65b1" />

<img width="2161" height="1095" alt="step-3" src="https://github.com/user-attachments/assets/8c3955e7-7f22-431e-82cf-5fbb64312ffa" />

## Walkthrough

### Step 1: The Login Page — and Its Comments

Browse to the target IP: a **login page**. Before trying any credentials, view the page source. In the **HTML comments**, the developers have left the login: username **`dj`**, password **`dj`**.

<img width="2335" height="1304" alt="step-4" src="https://github.com/user-attachments/assets/76a465a8-da0a-4330-ad8d-123dcdee5b29" />

<img width="1995" height="679" alt="step-5" src="https://github.com/user-attachments/assets/7acee531-0de5-4a1c-ba92-ec4d781f605d" />

### Step 2: Log In

Enter the commented credentials. They work — the application opens.

<img width="2391" height="1121" alt="step-6" src="https://github.com/user-attachments/assets/182a18ab-3658-44e4-ba8e-46925f5164c3" />

<img width="2828" height="1187" alt="step-7" src="https://github.com/user-attachments/assets/054515bc-82b5-467a-b453-8c5a875cc497" />

### Step 3: Export the App's Data — a YAML File

Inside, use the app's **export** feature and download what it produces, then open the file and study it: the export is a **YAML** document, and its structure shows exactly how the application represents its data — which matters if the application ever *reads YAML back in*.

<img width="2725" height="1383" alt="step-8" src="https://github.com/user-attachments/assets/8dae5db5-5dbc-4280-aaf7-79f390d78fbd" />

<img width="1739" height="797" alt="step-9" src="https://github.com/user-attachments/assets/b41c76f4-bcfc-4857-85fd-9b401aab48c6" />

### Step 4: Check the Technology Stack

Fingerprint the stack (headers, page behaviour, the export format itself). Two facts combine into the attack: the backend is **Python**, and the data format is **YAML** — which points at **PyYAML**, whose default `load` is famously unsafe against crafted documents.

<img width="2719" height="700" alt="step-10" src="https://github.com/user-attachments/assets/02ed0526-aa3b-49cf-bf6a-db600db0d3d7" />

### Step 5: Confirm YAML Deserialization RCE

PyYAML's unsafe loader honours tags like `!!python/object/apply:os.system`, which call a Python function *during parsing*. Submit a test document through the app's YAML-handling feature with a harmless command embedded in that shape. The server processes the document — and the command's **output comes back**. Arbitrary system commands are executing on the server: the deserialization vulnerability is confirmed, no shell needed yet.

<img width="1659" height="1254" alt="step-11" src="https://github.com/user-attachments/assets/c4a095a2-1c99-43a2-9a73-57f35ca1b83f" />

### Step 6: Reverse Shell Through the YAML Payload

Set up the listener on Kali:

```bash
nc -nlvp 4444
```

<img width="1855" height="439" alt="step-12" src="https://github.com/user-attachments/assets/84b7b881-2673-46c2-a255-42374af15168" />

Confirm Kali's tunnel IP for the callback:

```bash
ip a
```

<img width="2573" height="1491" alt="step-13" src="https://github.com/user-attachments/assets/4cfe629f-8cb9-447c-a66b-6d682fbfb1de" />

Now send the full payload — the same `python/object/apply` construction, this time carrying a reverse-shell command aimed at the listener:

<img width="1705" height="890" alt="step-14" src="https://github.com/user-attachments/assets/7b7c4cb2-9db5-41b5-a77d-cddc4c0b51a6" />

The listener catches the connection: a **shell on the target** as the application user.

<img width="2707" height="681" alt="step-15" src="https://github.com/user-attachments/assets/99dcaee3-c108-48f2-abd1-c40827a3e239" />

### Step 7: The User Flag

Root's directory is **access denied** — expected for this account. Go back to the user's **home directory** instead: the **flag** is there.

<img width="1459" height="1051" alt="step-16" src="https://github.com/user-attachments/assets/bbcb0bf3-a259-46da-b181-dfaaae238032" />

<img width="2332" height="1102" alt="step-17" src="https://github.com/user-attachments/assets/d4f7de24-a32e-4eff-b816-5cacc1a70e96" />

### Step 8: Read the Source — a Password on a Command Line

With local access, review the application's source code. One section stands out immediately: the jukebox **service is started with its password supplied as a command-line argument**. That is a leak by design — on Linux, every process's full command line is visible to every other user through the process list (`ps`, `/proc/<pid>/cmdline`). Secrets do not belong in argv.

<img width="2833" height="496" alt="step-18" src="https://github.com/user-attachments/assets/01576b6b-3567-4dde-86b4-26f7f0a36ba0" />

### Step 9: Harvest the Password from `ps`

Inspect the running processes, filtered for the jukebox service:

```bash
ps aux | grep -i jukebox
```

The service's command line prints in full — **password included** (the value is visible in the screenshot and is not repeated here). Any local user, including the low-privilege shell from Step 6, can read it. That recovered service password is the room's final key.

<img width="2836" height="573" alt="step-19" src="https://github.com/user-attachments/assets/07297bae-8597-4123-9668-90687690d1c4" />

<img width="1886" height="492" alt="step-20" src="https://github.com/user-attachments/assets/e675a7c5-05d3-478d-943e-56890c642c39" />

## Key Takeaways

- Page source is part of the application you are attacking: working credentials sat in HTML comments on the login page itself. View-source is step one, not an afterthought.
- Unsafe deserialization is code execution with extra steps. PyYAML's `yaml.load` on attacker data — like Java's and PHP's equivalents — runs the object graph the document describes, including `os.system`. If an app imports a rich data format from users, the parser's safety mode *is* the security boundary.
- An export feature is an oracle for the import format: the YAML the app produced revealed exactly the structure to weaponise on the way back in.
- Command-line arguments are public on a multi-user system. Passwords passed in argv are published to `ps` and `/proc` for every local account — services must take secrets from files, environment or a vault, never from the command line.
- Denied `/root` is not a dead end: home directories, application source and the process list are the standard post-shell enumeration trio, and here they paid out twice.

## Remediation

Remove credentials and developer notes from client-side code entirely, and give demo accounts no path to production data. Parse YAML only with a safe loader (`yaml.safe_load`) and treat every deserialization format as untrusted input — prefer plain data formats (JSON) at trust boundaries. Run application services under dedicated low-privilege accounts, and pass secrets via protected files, environment variables or a secrets manager — never as command-line arguments, which the process list exposes. Finally, keep flags and sensitive data out of broadly readable home directories, and audit what a low-privilege local user can enumerate: processes, source trees and exports included.
