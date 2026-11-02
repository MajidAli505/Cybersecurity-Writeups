# Password Reset Poisoning via Middleware

**Platform:** PortSwigger Web Security Academy
**Category:** Authentication Vulnerabilities
**Difficulty:** Practitioner
**Status:** ✅ Solved

## Overview

Password reset poisoning occurs when an application builds the reset link in the password-reset email using a host value that an attacker can control. In this lab, middleware in front of the application honors the `X-Forwarded-Host` header, and the reset email's domain is generated from it. An attacker can therefore request a password reset for the victim while pointing the link's domain at their own exploit server. When the victim clicks the link, their browser connects to the attacker's server — handing over the secret reset token.

## Lab Objective

Log in to Carlos's account by stealing his password reset token through a poisoned reset email. Carlos carelessly clicks any link he receives by email.

**Provided resources:** your credentials `wiener:peter`; emails to your account can be read via the email client on the exploit server.

<img width="2145" height="699" alt="step-1" src="https://github.com/user-attachments/assets/1d9c14ac-6782-4821-ba62-c1dc29328b7a" />

## Walkthrough

### Step 1: Examine the Normal Reset Flow

With Burp running, investigate the password reset functionality using your own account. Observe that the reset email contains a link with a unique reset token — and that the link's domain is generated dynamically rather than hard-coded.

### Step 2: Send the Reset Request to Repeater

Send the `POST /forgot-password` request to Burp Repeater. Testing shows that the `X-Forwarded-Host` header is supported: whatever host it names is used to build the domain in the reset email link, so the link can be pointed at an arbitrary domain.

### Step 3: Note Your Exploit Server Domain

Open the exploit server and make a note of its URL — this is the domain that will receive the victim's click (and his token).

### Step 4: Poison the Reset Request for Carlos

In Repeater, add the header with your exploit server as its value:

```text
X-Forwarded-Host: YOUR-EXPLOIT-SERVER-ID.exploit-server.net
```

Then change the `username` parameter in the request body to `carlos` and send the request. The application emails Carlos a reset link whose token is genuine, but whose domain is your exploit server.

### Step 5: Collect the Token from the Access Log

Carlos clicks the link in his email. His browser requests the reset path from your exploit server, so open the exploit server's access log: the `GET /forgot-password` entry contains the victim's reset token as a query parameter. Make a note of the token.

### Step 6: Build a Valid Reset URL with the Stolen Token

Go to your own email client and copy **your own** valid password reset link (the one pointing at the real lab domain — not the poisoned one). Paste it into the browser and replace the value of its `temp-forgot-password-token` parameter with the token stolen from Carlos.

### Step 7: Set a New Password and Log In

Load the modified URL and set a new password for Carlos's account. Then log in to Carlos's account using the new password to solve the lab.

## Reference Screenshot

<img width="2104" height="1503" alt="step-2" src="https://github.com/user-attachments/assets/bd854ed8-65e0-4748-812f-8cca04f8df50" />

## Key Takeaways

- Reset links must be built from a fixed, server-side configured base URL. Taking the host from request headers (`Host`, `X-Forwarded-Host`) lets attackers redirect victims — and their tokens — to attacker-controlled infrastructure.
- Middleware and proxies that pass forwarding headers through must be configured so those headers are only trusted from known proxies, and applications should ignore them when generating absolute URLs.
- The token itself was strong; the attack never broke it. It simply arranged for the token to be delivered to the wrong place — a reminder that the delivery channel is part of the security design.

## Remediation

- Hard-code the application's public domain for email links, or use an allowlist; never derive it from client-controlled headers.
- Configure middleware to strip or overwrite `X-Forwarded-Host` from untrusted sources.
- Keep reset tokens single-use and short-lived, and warn users to verify the domain before entering a new password.
