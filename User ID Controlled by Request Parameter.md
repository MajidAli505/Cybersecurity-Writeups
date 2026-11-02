# User ID Controlled by Request Parameter

**Platform:** PortSwigger Web Security Academy
**Category:** Access Control
**Difficulty:** Apprentice
**Status:** ✅ Solved

## Overview

When a page decides whose data to show based on an identifier in the request, that identifier must be checked against the logged-in user. This lab's account page takes the user ID straight from a URL parameter and never compares it with the session. The result is horizontal privilege escalation: any logged-in user can view any other user's account page — including the secrets displayed on it — by changing one word in the address bar.

## Lab Objective

This lab has a horizontal privilege escalation vulnerability on the user account page. Obtain the API key for the user `carlos` and submit it as the solution.

**Provided resources:** your credentials `wiener:peter`.

<img width="2168" height="892" alt="step-1" src="https://github.com/user-attachments/assets/551dd51f-fe90-4f76-918b-463017223004" />

## Walkthrough

### Step 1: Log In and Inspect Your Own Account Page

Log in using the supplied credentials and open your account page. Two things stand out. The URL names the account being displayed:

```http
GET /my-account?id=wiener
```

And the page itself displays sensitive data — your username and your API key. Everything on this page is selected by that `id` parameter.

<img width="3634" height="1890" alt="step-2" src="https://github.com/user-attachments/assets/a4353d49-8574-4efd-a59b-b66937bc4210" />

<img width="3224" height="1567" alt="step-3" src="https://github.com/user-attachments/assets/b9490b32-9a0b-425b-8113-3792933e3831" />

### Step 2: Change the ID to carlos

Change the parameter in the URL from your own username to the target's:

```http
GET /my-account?id=carlos
```

The page loads without complaint — still under your session — but it now shows `carlos`'s account, including his API key. The server authenticated you as `wiener`, then served whoever the parameter asked for; at no point did it check that the two matched.

<img width="3247" height="1590" alt="step-4" src="https://github.com/user-attachments/assets/307af2d2-bbad-4c89-a150-964a7df8ac5f" />

### Step 3: Submit the API Key

Copy the API key from `carlos`'s account page and submit it using the lab's "Submit solution" button. The lab is solved.

<img width="2884" height="1017" alt="step-5" src="https://github.com/user-attachments/assets/6d1850e2-77f5-4fcf-98fe-c066559cb36b" />

<img width="3150" height="1509" alt="step-6" src="https://github.com/user-attachments/assets/ac0dcd4d-aab1-4ce2-9eeb-d6df0ce59de8" />

## Key Takeaways

- Authentication answers "who is asking?"; it does not answer "what may they see?". This page had the first and skipped the second.
- A user-controlled identifier selecting records is an IDOR pattern: if the server does not bind the requested object to the session's identity, every user can read every record.
- The exposed data mattered: an API key on the page turned a read-only privacy leak into full credential disclosure for the victim account.

## Remediation

Never trust a client-supplied user ID to select account data. Derive the account from the authenticated session server-side; where an identifier is genuinely needed (for example, admin tools), verify on every request that the session is authorized for that specific record. Apply the check at the data-access layer so no route can forget it.
