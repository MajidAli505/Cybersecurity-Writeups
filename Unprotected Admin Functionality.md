# Unprotected Admin Functionality

**Platform:** PortSwigger Web Security Academy  
**Category:** Access Control  
**Difficulty:** Apprentice  
**Status:** ✅ Solved

## Overview

The most basic access control failure is having none at all. Some applications rely entirely on the admin interface being hard to find: the link is never shown to ordinary users, so the developers assume nobody will request the URL directly. But a URL is not a credential. If the server performs no authentication or authorization check when the admin page is requested, then anyone who discovers the path — by guessing, wordlists, or a hint left in a public file — gets full administrative control.

## Lab Objective

Find this site's hidden admin panel and use it to delete the user `carlos`.

<img width="2178" height="782" alt="step-1" src="https://github.com/user-attachments/assets/e1627fb4-d2b8-485f-a97c-a1902f741cc7" />

## Walkthrough

### Step 1: Browse the Application

Open the lab and look around. Nothing in the normal interface links to any administration area — as far as a regular visitor can see, it doesn't exist.

<img width="3511" height="1677" alt="step-2" src="https://github.com/user-attachments/assets/4d8506a4-1448-4cf0-b752-317a4f76d6ae" />

### Step 2: Check robots.txt

Fetch `/robots.txt`. It contains a disallow entry that names the sensitive path outright:

```text
User-agent: *
Disallow: /administrator-panel
```

The file meant to steer crawlers away from the admin panel also advertises it to every human who reads it.

<img width="1910" height="399" alt="step-3" src="https://github.com/user-attachments/assets/f642dcd3-a76a-4af1-8c85-9bfee9304a01" />

### Step 3: Visit the Admin Panel Directly

Browse to the disclosed path:

```text
/administrator-panel
```

The page loads immediately — no login, no role check, nothing. It presents the user administration interface with the full user list and a **Delete** action next to each account.

<img width="2359" height="1009" alt="step-4" src="https://github.com/user-attachments/assets/ef3c05f0-56e8-4f3d-813f-6bb401c7f98c" />

### Step 4: Delete Carlos

Click **Delete** next to `carlos`. The panel confirms the user was deleted successfully, and the lab is solved — an unauthenticated visitor just performed an administrator-only action.

<img width="3348" height="1177" alt="step-5" src="https://github.com/user-attachments/assets/b9fb5183-78b1-430c-ac86-4c2c2707ac13" />

## Key Takeaways

- Security through obscurity is not access control. An unlinked URL stops nobody; every sensitive function must verify the caller's identity and role on the server, on every request.
- `robots.txt` is public by design. Listing sensitive paths in it maps them for attackers; it should never be treated as a hiding mechanism.
- Always probe for common admin paths during testing (`/admin`, `/administrator-panel`, and similar), and check `robots.txt` early — it frequently names them.

## Remediation

- Enforce authentication and role-based authorization on the server for every administrative page and action, regardless of whether the URL is linked or "secret".
- Keep genuinely sensitive paths out of `robots.txt`; if crawlers must be kept away, combine minimal disclosure with real access controls.
- Deny access by default and audit the application for unprotected endpoints, including ones no longer linked from the UI.
