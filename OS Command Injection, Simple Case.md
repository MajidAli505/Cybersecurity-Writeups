# OS Command Injection, Simple Case

**Platform:** PortSwigger Web Security Academy
**Category:** OS Command Injection
**Difficulty:** Apprentice
**Status:** ✅ Solved

## Background

OS command injection occurs when an application builds a shell command out of user-supplied input and executes it without sanitization. The input stops being data and becomes part of the command line: shell metacharacters such as `|` let an attacker append an entirely new command of their choosing. This is the simplest form of the bug — the application runs the command and helpfully returns its raw output in the HTTP response, so the result of the injected command is immediately visible.

## Lab Objective

This lab contains an OS command injection vulnerability in the product stock checker. The application executes a shell command containing the user-supplied product and store IDs, and returns the raw output in its response. Execute the `whoami` command to determine the name of the current user.

<img width="2219" height="946" alt="step-1" src="https://github.com/user-attachments/assets/8378eee2-486e-429f-87fc-e1f6a415143e" />

## Walkthrough

### Step 1: Find the Stock Checker

Open any product on the store. At the bottom of the product page is a **Check stock** feature — pick a store and it reports how many units are in stock. That lookup is where the vulnerable command runs.

<img width="3115" height="1408" alt="step-2" src="https://github.com/user-attachments/assets/320ee38a-07d0-4af0-bbb9-beafa98d566f" />

<img width="3115" height="1350" alt="step-3" src="https://github.com/user-attachments/assets/691dc299-0cb2-491a-bbb9-def42e4ec056" />

### Step 2: Capture the Request

Click Check stock with Burp running and capture the request, then send it to Repeater:

```http
POST /product/stock HTTP/2
Host: <lab host>

productId=1&storeId=1
```

Both IDs are placed into the shell command the application executes — and `storeId` is the one to attack.

<img width="2298" height="1048" alt="step-4" src="https://github.com/user-attachments/assets/18028bf8-17f3-471c-9f18-702e4fd97aba" />

### Step 3: Inject a Second Command

Append a shell pipe and a command of your own to the `storeId` value:

```http
productId=1&storeId=1|whoami
```

The application concatenates this into its shell command, which now reads (in effect) *"look up stock for store 1, then also run `whoami`"*. The shell obliges, and because this application returns the command's raw output in the response, the reply contains the output of `whoami` — the name of the user the server is running as. The lab is solved.

<img width="2629" height="1149" alt="step-5" src="https://github.com/user-attachments/assets/4af43d5f-8a06-4501-baad-2b837a061f9" />

<img width="3072" height="1662" alt="step-6" src="https://github.com/user-attachments/assets/61bde78f-8931-4500-b595-8176fcbcf658" />

## Key Takeaways

- Any input that reaches a shell command string is code, not data. The pipe character is the classic separator, but `&`, `;`, `&&`, and backticks serve the same purpose.
- When output comes back in the response, exploitation is trivially confirmable — a harmless command like `whoami` proves code execution without touching anything.
- The vulnerable surface is often a mundane utility feature (a stock checker, a ping tool, a converter), not the headline functionality.

## Remediation

Avoid invoking a shell with user input at all: call executables directly with argument arrays (no shell interpretation), or better, use a native API for the task. Where a shell command is unavoidable, validate input against a strict allowlist (for IDs, require integers) and reject anything containing shell metacharacters — validation must happen server-side, on every value that reaches the command.
