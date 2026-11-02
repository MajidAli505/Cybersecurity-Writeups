# User Role Controlled by Request Parameter

**Platform:** PortSwigger Web Security Academy
**Category:** Access Control
**Difficulty:** Apprentice
**Status:** ✅ Solved

## Overview

A role is a fact about a user that the server must determine for itself. This lab's application instead lets the client *declare* it: alongside the session cookie, the browser carries an `Admin` cookie whose value states whether the user is an administrator — and the server simply believes it. Cookies are stored and sent by the client, so anything kept in one is fully editable by the user. Asking the client "are you an admin?" and trusting the answer is not an access control at all; it is an honor system.

## Lab Objective

Use the forgeable role cookie to reach the admin panel at `/admin` and delete the user `carlos`.

**Provided resources:** your credentials `wiener:peter`.

<img width="2157" height="879" alt="step-1" src="https://github.com/user-attachments/assets/9d73aaf6-a173-4f59-a0db-cd8c45ea23da" />

## Walkthrough

### Step 1: Log In and Browse

Log in as `wiener` and browse the site with Burp running, so your traffic is captured in the proxy history.

<img width="3411" height="1887" alt="step-2" src="https://github.com/user-attachments/assets/16255599-2b42-4ec0-a7c5-75c31dd0eb2b" />

<img width="2189" height="1383" alt="step-3" src="https://github.com/user-attachments/assets/e6dc0de6-3df9-4af0-85a8-889cc07cd476" />

### Step 2: Inspect the /admin Request in Burp

In the proxy's HTTP history, find the request to `GET /admin`. The response is `401 Unauthorized` — but look at the request's cookies:

```http
Cookie: Admin=false; session=<your session value>
```

The application itself announces how it tracks your role: a plaintext cookie, currently set to `false`, that you are free to change.

<img width="2600" height="1802" alt="step-4" src="https://github.com/user-attachments/assets/2499841b-60e6-44e3-b9b5-960b4431618c" />

### Step 3: Forge the Cookie in Repeater

Send the `/admin` request to Burp Repeater and edit the cookie, flipping the role flag:

```http
Cookie: Admin=true; session=<your session value>
```

<img width="2548" height="1042" alt="step-5" src="https://github.com/user-attachments/assets/7c5e289f-9557-4146-9ee6-be469311f0f1" />

### Step 4: Receive the Admin Panel

Send the modified request. This time the response contains the full admin panel — the user list showing `wiener` and `carlos`, each with a Delete option. The server never checked who you are; it only read the value you handed it. (Opening the response in the browser makes it easy to click through.)

<img width="2510" height="1440" alt="step-6" src="https://github.com/user-attachments/assets/fa2bdfa9-0db8-476d-845b-6754fc4df6c1" />

<img width="3320" height="1042" alt="step-7" src="https://github.com/user-attachments/assets/22a27faa-a502-4d45-b4f3-39df4b01ed03" />

### Step 5: Delete Carlos

From the admin panel, delete the user `carlos`. The panel confirms the deletion and the lab is solved — full administrative access granted by editing one word in a cookie.

<img width="3156" height="1209" alt="step-8" src="https://github.com/user-attachments/assets/84d772c7-726d-449e-aadf-bbe6910455be" />

## Key Takeaways

- Any role or privilege information stored client-side (cookies, hidden fields, local storage) is attacker-controlled. Authorization decisions must be derived server-side from the authenticated session.
- A `true`/`false` flag is especially fragile: no guessing, cracking, or tooling is required — just curiosity and a proxy.
- When testing, inventory every cookie and parameter your client sends; names like `Admin`, `role`, or `isAdmin` are prime targets.

## Remediation

- Store roles in server-side session data or a database, keyed by the authenticated user; never accept privilege claims from the request.
- If role data must travel with the client (for example in a token), sign it and verify the signature server-side so tampering is detected.
- Re-check authorization on the server for every sensitive action, independent of what the UI shows or what the request asserts.
