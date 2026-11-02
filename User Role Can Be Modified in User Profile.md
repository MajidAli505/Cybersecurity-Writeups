# User Role Can Be Modified in User Profile

**Platform:** PortSwigger Web Security Academy
**Category:** Access Control
**Difficulty:** Apprentice
**Status:** ✅ Solved

## Overview

Profile-update endpoints are only supposed to change the fields the user is offered — here, an email address. This lab's endpoint is built differently: it accepts a JSON body and applies the fields it finds to the user record without checking which fields a normal user should be allowed to touch. Worse, its response helpfully returns the full user object, including a `roleid` field the interface never shows. That combination — a writable hidden field, disclosed by the application itself — is a classic mass-assignment flaw: if the server binds whatever you send, you can send yourself a promotion.

## Lab Objective

Reach the admin panel at `/admin`, which is only accessible to logged-in users with a `roleid` of 2, and use it to delete the user `carlos`.

**Provided resources:** your credentials `wiener:peter`.

<img width="2199" height="895" alt="step-1" src="https://github.com/user-attachments/assets/91f4df3b-03c3-4e89-8af7-2431c03a6fa1" />

## Walkthrough

### Step 1: Log In and Open Your Account Page

Log in using the supplied credentials and access your account page.

<img width="3052" height="1916" alt="step-2" src="https://github.com/user-attachments/assets/1ccb51bd-9b49-48b1-9e8f-c0a8518d1bb8" />

<img width="2236" height="1414" alt="step-3" src="https://github.com/user-attachments/assets/29004bc8-b187-49ad-9be4-43f1617bcd14" />

### Step 2: Update Your Email and Read the Response

Use the provided feature to update the email address associated with your account, and watch the request in Burp. The `POST /my-account/change-email` request carries only the new email — but the response returns your whole profile as JSON, including a field the page never mentions:

```json
{
    "username": "wiener",
    "email": "<your new email>",
    "apikey": "<your API key>",
    "roleid": 1
}
```

Your current role is `1`. The admin panel requires `2` — and the application has just told you the exact name of the field that controls it.

<img width="2721" height="1545" alt="step-4" src="https://github.com/user-attachments/assets/f534a0b9-ac8e-460d-9a32-f933422713d6" />

### Step 3: Add the Role Field in Burp Repeater

Send the email submission request to Burp Repeater, add `"roleid": 2` into the JSON in the request body, and resend it:

```json
{
    "email": "<your new email>",
    "roleid": 2
}
```

<img width="2230" height="1041" alt="step-5" src="https://github.com/user-attachments/assets/1a9dd4d8-d654-44c7-b8eb-1c7748056174" />

### Step 4: Confirm the New Role

Observe that the response shows your `roleid` has changed to 2. The server accepted a field it was never offered in the interface and applied it to your account without any authorization check — you are now an administrator as far as the application is concerned.

<img width="2633" height="1353" alt="step-6" src="https://github.com/user-attachments/assets/cc5e398d-4821-4161-adc9-ca2510be10a1" />

<img width="2765" height="1106" alt="step-7" src="https://github.com/user-attachments/assets/b947941c-478e-477a-844a-b532e5ea426e" />

### Step 5: Open the Admin Panel and Delete carlos

Browse to `/admin`. With `roleid` 2 on your account, the panel opens and lists the users. Delete `carlos` — the page confirms "User deleted successfully" and the lab is solved.

<img width="2520" height="1255" alt="step-8" src="https://github.com/user-attachments/assets/bd4d4a08-c2a9-441a-adbf-7c1a7dee66d5" />

<img width="3053" height="956" alt="step-9" src="https://github.com/user-attachments/assets/6a8ec86a-fe24-4cd6-9313-93a3a45d5a00" />

<img width="3133" height="1031" alt="step-10" src="https://github.com/user-attachments/assets/0dcd78fc-f72e-43ae-9a26-994633a2e65e" />

## Key Takeaways

- An update endpoint that binds request fields directly to a data object lets users write any property they can name — including ones, like a role, that decide their privileges.
- Responses that return the full object are reconnaissance for free: the field name and current value needed for the attack were handed over by a routine email change.
- The check failed on the write side, not the read side: `/admin` correctly required `roleid` 2, but nothing stopped an ordinary user from *setting* it.

## Remediation

Bind user input to an explicit allowlist of editable fields (email, display name, and so on) instead of the underlying user object, and never accept privilege-bearing fields such as role identifiers from client requests. Role changes should live on a separate, admin-only endpoint with its own authorization check. Returning only the fields the client needs also removes the free disclosure that made this attack obvious.
