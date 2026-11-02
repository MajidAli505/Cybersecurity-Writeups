# Password Brute-Force via Password Change

**Platform:** PortSwigger Web Security Academy  
**Category:** Authentication Vulnerabilities  
**Difficulty:** Practitioner  
**Status:** ✅ Solved

## Overview

Login pages are the obvious brute-force target, so they tend to get lockouts and rate limits. Password **change** pages verify the current password using the same check, yet they are often forgotten by those defenses. This lab's change-password functionality accepts the target username as a request parameter and has no lockout — and its error messages differ depending on whether the current password was right. That turns the change-password form into an unlimited password-guessing oracle for any account.

## Lab Objective

Use the candidate password list to brute-force Carlos's current password through the password change functionality, then access his **My account** page.

**Provided resources:** your credentials `wiener:peter`, the victim's username `carlos`, and a list of candidate passwords.

<img width="2206" height="896" alt="step-1" src="https://github.com/user-attachments/assets/b67d8cc4-9c9e-43fd-b6e4-d16756740d4b" />

## Walkthrough

### Step 1: Explore the Change-Password Form with Your Own Account

Log in and open the password change form. It asks for the current password and the new password twice. Submit it with your **correct** current password but two **different** new passwords, and observe the response:

```text
New passwords do not match
```

This error is the key to the attack: it is only shown **after** the current password has been verified successfully, so its presence tells us the current-password guess was correct.

### Step 2: Send the Request to Burp Intruder

Capture that change-password request and send it to Burp Intruder. In the request, change the `username` field to `carlos` — the form trusts this client-supplied value instead of relying only on the session. Add a payload position to the `current-password` parameter, and keep the two new-password fields set to two different values (so the "do not match" error remains our success signal).

### Step 3: Load the Candidate Passwords

Load the provided list of candidate passwords as the payload set for the `current-password` position.

### Step 4: Add a Grep Match Rule for the Success Signal

Open Intruder's **Settings** and add a grep match rule for the string:

```text
New passwords do not match
```

Start the attack. Wrong guesses fail the current-password check first and return a different response; only the correct password gets far enough to trigger the "do not match" error.

### Step 5: Identify Carlos's Password

When the attack finishes, exactly one response contains the `New passwords do not match` message. The payload used in that request is Carlos's current password — make a note of it.

### Step 6: Log In as Carlos

Log out of your own account and log in with the username `carlos` and the password identified in Step 5. Open **My account** to solve the lab.

## Key Takeaways

- Any functionality that verifies a password — login, password change, sensitive-action confirmation — is a brute-force surface and needs the same rate limiting and lockout protection as the login page.
- Trusting a client-supplied `username` field in the change-password request lets an attacker aim the attack at arbitrary accounts instead of only their own.
- Distinct error messages create oracles: here, reaching the "new passwords do not match" error proved the current password was correct, which is all an attacker needs.

## Remediation

- Derive the account for a password change from the authenticated session, never from a request parameter.
- Apply the same throttling, lockout, and monitoring to password-change attempts as to logins.
- Validate that the new passwords match **before** (or independently of) verifying the current password, and use generic error responses that do not reveal which check failed.
