# User ID Controlled by Request Parameter with Password Disclosure

**Platform:** PortSwigger Web Security Academy
**Category:** Access Control
**Difficulty:** Apprentice
**Status:** ✅ Solved

## Overview

Some account pages prefill the "change password" form with the user's existing password, rendered in a masked input. Masking is only cosmetic — the full password sits in the input's value, and therefore in the page's response body. Combine that design with an account page that serves whichever user an `id` parameter names, and a low-privilege user can harvest the administrator's actual password: not a token, not a hash, the reusable credential itself.

## Lab Objective

This lab has a user account page that contains the current user's existing password, prefilled in a masked input. Retrieve the administrator's password, then use it to delete the user `carlos`.

**Provided resources:** your credentials `wiener:peter`.

<img width="2191" height="1026" alt="step-1" src="https://github.com/user-attachments/assets/6b2ad704-2fa0-4460-9795-0c5f13b4e91c" />

## Walkthrough

### Step 1: Log In and Inspect the Password Field

Log in using the supplied credentials and open your account page (`/my-account?id=wiener`). Alongside the email form there is a password field, already filled with masked dots — your current password, prefilled by the application.

<img width="3099" height="1873" alt="step-2" src="https://github.com/user-attachments/assets/30f43c44-1e97-4f0c-a9aa-504b30ce0c07" />

<img width="2843" height="1489" alt="step-3" src="https://github.com/user-attachments/assets/1d85eb85-a97b-4fff-9745-2af0cf9b4e8a" />

### Step 2: Request the Administrator's Account Page

Change the ID parameter to `administrator`:

```http
GET /my-account?id=administrator
```

His account page loads under your session. His password field is also prefilled — and also masked. In the browser, you see only dots.

<img width="3286" height="1782" alt="step-4" src="https://github.com/user-attachments/assets/d356ff0a-7e83-4920-b66c-9d9041031e4b" />

<img width="3036" height="1355" alt="step-5" src="https://github.com/user-attachments/assets/2d22c1b0-2fa6-4972-b7db-d15ff7493557" />

### Step 3: Read the Password in Burp

Capture the request in Burp and read the response body. The masked input's value is transmitted in full: the administrator's plaintext password is sitting in the HTML of the page you were just served. The dots were never protection — they were presentation.

<img width="3118" height="1924" alt="step-6" src="https://github.com/user-attachments/assets/865e07cf-2832-430e-aa4b-db12cd6c5de5" />

### Step 4: Log In as Administrator and Delete carlos

Log out, log back in as `administrator` with the recovered password, open the admin panel, and delete `carlos`. The page confirms "User deleted successfully" and the lab is solved.

<img width="2806" height="951" alt="step-7" src="https://github.com/user-attachments/assets/ce5fd118-9031-4620-b178-bd5b8045fae7" />

<img width="3007" height="1462" alt="step-8" src="https://github.com/user-attachments/assets/74fcade0-929a-4212-80af-5167f1ed0578" />

<img width="3188" height="1830" alt="step-9" src="https://github.com/user-attachments/assets/d4c13830-fbca-46cd-bc06-e2514324c5a9" />

<img width="3039" height="1001" alt="step-10" src="https://github.com/user-attachments/assets/40808455-a7b5-4b07-bc3f-1343830fd865" />

<img width="3123" height="1180" alt="step-11" src="https://github.com/user-attachments/assets/3b87c5d3-4dcf-4bd9-a3a3-429d0372b70d" />

## Key Takeaways

- A masked password field still ships the password to whoever receives the page. Anything in a response body is disclosed, regardless of how the browser renders it.
- The two flaws compound: password prefill alone exposes your own credential to anything that can read your traffic or page source; the missing ownership check extends that exposure to every account, including the administrator's.
- Disclosed passwords are worse than disclosed data: they are reusable, so the impact jumps from viewing one page to full account takeover.

## Remediation

Never send a user's existing password back to the client — password fields in account pages should start empty, and changing a password should require the current one to be typed in. Passwords should be stored as salted hashes, which also makes prefill impossible by design. And as with the rest of this series, the account page must verify that the requested record belongs to the authenticated session.
