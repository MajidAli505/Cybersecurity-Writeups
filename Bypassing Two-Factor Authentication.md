# Bypassing Two-Factor Authentication

**Platform:** PortSwigger Web Security Academy  
**Category:** Authentication Vulnerabilities  
**Difficulty:** Apprentice  
**Status:** ✅ Solved

## Overview

This lab demonstrates a two-factor authentication (2FA) bypass caused by a flawed implementation. The vulnerability is not about cracking the verification code — it is about exploiting a logic mistake: the application creates a fully authenticated session as soon as the username and password are verified, before the second factor has been completed.

## Background

In a correct 2FA flow, protected pages should only be accessible after **both** factors are completed:

```text
Is the user authenticated?
        │
        ▼
Has the user completed 2FA?
        │
        ▼
Allow access
```

In the vulnerable implementation, however, the developer only checks whether the user is logged in:

```text
if (logged_in == true)
    allow_access();
```

instead of the secure logic:

```text
if (logged_in == true && two_factor_verified == true)
{
    allow_access();
}
else
{
    redirect_to_2FA();
}
```

Because the session already exists after the password step, an attacker who knows the victim's password can simply skip the verification page and browse directly to a protected URL such as `/my-account` — the application checks only the session, not whether 2FA was completed.

## Lab Objective

Log in to the victim's account without completing the 2FA verification step.

## Walkthrough

### Step 1: Explore the Flow with Your Own Account

Log in to your own account. Your 2FA verification code is sent to you by email — click the **Email client** button to access your emails and complete the login normally.

### Step 2: Note the Account Page URL

Go to your account page and make a note of its URL (`/my-account`).

### Step 3: Log Out and Log In as the Victim

Log out of your account, then log in using the victim's credentials.

### Step 4: Skip the Verification Step

When you are prompted for the verification code, do **not** enter one. Instead, manually change the URL in the address bar to navigate directly to `/my-account`.

The account page loads successfully, and the lab is solved — full account access was granted without the second factor ever being verified.

## Key Takeaways

- Verifying a password must not, by itself, create a session that grants access to protected resources when 2FA is enabled.
- The server must track the authentication state explicitly (password verified vs. 2FA completed) and enforce the completed state on every protected request.
- A redirect to the 2FA page is only a user-interface step; it is not a security control unless the backend enforces it.

## Remediation

- Only issue a fully privileged session after all authentication factors have been completed successfully.
- Store the 2FA completion state server-side and validate it on every request to a protected resource.
- Treat partially authenticated sessions as unprivileged until the final factor succeeds.
