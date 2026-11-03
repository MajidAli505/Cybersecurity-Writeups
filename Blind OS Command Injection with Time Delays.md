# Blind OS Command Injection with Time Delays

**Platform:** PortSwigger Web Security Academy
**Category:** OS Command Injection
**Difficulty:** Practitioner
**Status:** ✅ Solved

## Background

In a *blind* command injection, the injected command still executes — but its output never appears in the HTTP response, so there is nothing to read. Proof has to come from a side effect the attacker *can* observe. The most reliable one is **time**: inject a command that makes the server pause for a known duration (such as pinging a loopback address a fixed number of times). If the response takes exactly that long to arrive, the command ran — execution confirmed, without seeing a single byte of output.

## Lab Objective

This lab contains a blind OS command injection vulnerability in the feedback function: the application executes a shell command containing the user-supplied details, but the output is not returned in the response. Exploit the vulnerability to cause a 10-second delay.

<img width="2186" height="1137" alt="step-1" src="https://github.com/user-attachments/assets/f569c5a7-a7e4-4d97-8697-0aa85eb0de7f" />

## Walkthrough

### Step 1: Submit the Feedback Form

Open the site's feedback page and submit the form normally, with Burp proxying, so the request is captured.

<img width="3100" height="1772" alt="step-2" src="https://github.com/user-attachments/assets/c44e25b7-17ef-4b9b-9a6f-e06374d9f657" />

<img width="2592" height="1672" alt="step-3" src="https://github.com/user-attachments/assets/5de628c3-117f-41b3-addf-aa3b50a7879a" />

### Step 2: Intercept and Modify the Email Parameter

Intercept the submission request. The form fields — including `email` — are fed into a shell command on the server. Modify the `email` parameter to append an injected command using the shell's OR operator, so the injected command runs and holds the connection open for about ten seconds:

```text
email=<your-email>||ping+-c+10+127.0.0.1||
```

(URL-encoded in the request body, this is `||ping+-c+10+127.0.0.1||` attached to the address.) The `ping -c 10` sends ten ICMP packets at one-second intervals — the shell cannot finish until the ping does.

<img width="2631" height="1158" alt="step-4" src="https://github.com/user-attachments/assets/7d7e6296-2d5e-4302-8128-7c7e25b5b749" />

<img width="2478" height="1254" alt="step-5" src="https://github.com/user-attachments/assets/33163b1f-2597-4df6-9af6-59fccba38d6e" />

### Step 3: Observe the Delay

Forward the request and watch the clock: the response takes roughly **10 seconds** longer than a normal submission. Nothing in the response body reveals the command's output — the delay *is* the output. Code execution is proven, and the lab is solved.

<img width="3053" height="1485" alt="step-6" src="https://github.com/user-attachments/assets/5da6886d-49dd-4a2d-a862-88b8615aec9e" />

## Key Takeaways

- Blind does not mean safe or unexploitable — it only changes how you observe success. Timing is an oracle: the response duration leaks whether (and sometimes what) the server executed.
- `ping -c N` is the classic delay primitive because its runtime is predictable and it needs no inbound connectivity; `sleep N` works the same way where available.
- Establishing a baseline first (a normal submission's response time) is what turns "the site felt slow" into evidence.

## Remediation

The fix is the same as for any command injection: never build shell commands from user input — invoke programs directly with argument arrays or use native APIs, and strictly validate anything that must be passed (an email field should be validated as an email address, which alone excludes shell metacharacters). Server-side allowlist validation matters even where output is never displayed, because timing and out-of-band channels still expose execution.
