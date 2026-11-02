# Basic Password Reset Poisoning

**Platform:** PortSwigger Web Security Academy  
**Category:** Authentication Vulnerabilities  
**Difficulty:** Apprentice  
**Status:** ✅ Solved

## Overview

In this, the simplest form of password reset poisoning, the application builds the reset link in the password-reset email from the request's `Host` header. Since the `Host` header is supplied by the client, an attacker can request a password reset for the victim while naming their own exploit server as the host. The victim receives a genuine reset token — in a link pointing at the attacker's domain. One careless click delivers the token straight into the attacker's access log.

## Lab Objective

Log in to Carlos's account by stealing his password reset token through a poisoned reset email. Carlos carelessly clicks any link he receives by email.

**Provided resources:** your credentials `wiener:peter`; emails to your account can be read via the email client on the exploit server.

<img width="2195" height="760" alt="step-1" src="https://github.com/user-attachments/assets/871d76f0-20fa-4451-af4a-be11248da00c" />

## Walkthrough

### Step 1: Request a Reset for Your Own Account

Go to the login page and use the **Forgot your password?** functionality to request a password reset for your own account.

<img width="2528" height="1427" alt="step-2" src="https://github.com/user-attachments/assets/0ea74676-7c2e-49e5-9956-9b80856689f3" />

<img width="1908" height="536" alt="step-3" src="https://github.com/user-attachments/assets/65a97f76-d6fc-4d6a-808f-215b69703c44" />

### Step 2: Inspect the Reset Email

Open the email client on the exploit server. The reset email contains a link to set a new password, and the URL carries the secret as the `temp-forgot-password-token` query parameter.

<img width="2566" height="1566" alt="step-4" src="https://github.com/user-attachments/assets/a6476091-33b4-4a5e-b43a-dee9d0386554" />

### Step 3: Complete a Normal Reset

Follow the link, enter a new password of your choice, and confirm that the flow works end to end. This baseline matters: next, we change one variable at a time.

<img width="3124" height="1500" alt="step-5" src="https://github.com/user-attachments/assets/4d2bc6ef-ec89-4bef-90fa-99a1e7d56b40" />

<img width="2163" height="1206" alt="step-6" src="https://github.com/user-attachments/assets/ddac8815-18d8-4e98-bffe-3e88b9720f5e" />

### Step 4: Study the Reset Request in Burp

Review the HTTP history. The reset email is triggered by a `POST /forgot-password` request whose body contains the `username` of the account being reset. Send this request to Burp Repeater.

<img width="1796" height="1460" alt="step-7" src="https://github.com/user-attachments/assets/3c96b39c-18a8-471e-8f5e-ca51e163834f" />

### Step 5: Test Whether the Host Header Is Trusted

In Repeater, change the `Host` header to an arbitrary value and resend the request — the reset is still triggered successfully. Check the new email that arrives: the reset link inside it is built on your arbitrary host value instead of the site's real domain. The application is constructing email links from a client-controlled header.

<img width="3019" height="1536" alt="step-8" src="https://github.com/user-attachments/assets/847a8088-434f-41a2-910b-7f3066a329cd" />

### Step 6: Poison Carlos's Reset Email

Back in Repeater, set the `Host` header to the exploit server's domain and change the `username` body parameter to `carlos`. Send the request. Carlos now receives a reset email whose link points at your exploit server.

<img width="2985" height="849" alt="step-9" src="https://github.com/user-attachments/assets/d7b0bbab-663f-4730-9a81-733112294256" />

<img width="2555" height="1471" alt="step-10" src="https://github.com/user-attachments/assets/9c04836e-70db-4f58-9fcf-94ac4f78e579" />

### Step 7: Harvest the Token from the Access Log

When Carlos clicks the link, his browser requests the reset URL from your exploit server. Open the exploit server's access log and find the `GET /forgot-password` entry — its `temp-forgot-password-token` parameter is Carlos's genuine reset token. Note it down.

<img width="2066" height="923" alt="step-11" src="https://github.com/user-attachments/assets/99aa8a76-a3de-4d34-b3e9-8d474c6dd944" />

### Step 8: Use the Stolen Token on the Real Domain

From your email client, copy the genuine reset URL from your own first reset email. Visit that URL in the browser, but replace your token in it with Carlos's stolen token. Set a new password of your choosing for Carlos's account.

<img width="3122" height="1613" alt="step-12" src="https://github.com/user-attachments/assets/d7558a44-3b94-46c9-af79-3bc0ab458e83" />

<img width="3069" height="1276" alt="step-13" src="https://github.com/user-attachments/assets/e7fe35d3-25e3-459c-9b70-7048055f77d7" />

### Step 9: Log In as Carlos

Log in with the username `carlos` and the new password you just set. Carlos's **My account** page confirms the takeover and solves the lab.

<img width="3030" height="1548" alt="step-14" src="https://github.com/user-attachments/assets/2fd41c70-a9e2-417b-801c-208cf6149a4c" />

## Key Takeaways

- Email links must be generated from a fixed, server-configured domain. Deriving them from the `Host` header hands control of the link's destination to whoever sent the request.
- The reset token was not weak or predictable — the attack defeated the *delivery* of the token, not the token itself.
- Any user-triggered email containing links or tokens (resets, invitations, verification) deserves the same scrutiny as the reset flow here.

## Remediation

- Build absolute URLs in emails from server-side configuration, never from request headers.
- If the application sits behind a proxy, only trust forwarded host information from that proxy, and validate it against an allowlist.
- Keep tokens unguessable, single-use, and short-lived, and advise users to check the domain before acting on emailed links.
