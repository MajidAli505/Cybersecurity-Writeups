# Password Reset Broken Logic

**Platform:** PortSwigger Web Security Academy  
**Category:** Authentication Vulnerabilities  
**Difficulty:** Apprentice  
**Status:** ✅ Solved

## Overview

This lab's password reset flow looks secure at first glance: a reset link containing a token is emailed to the user, and the new password is submitted together with that token. The fatal flaw is that the server never actually validates the token when the new password is submitted — and the account being reset is identified by a `username` value in the request itself. Anyone can therefore reset **any** user's password without ever seeing their email.

## Lab Objective

Reset the victim `carlos`'s password without access to his email, then log in as him.

**Provided resources:** your own account (with access to its reset emails via the Email client) and the victim's username, `carlos`.

## Walkthrough

### Step 1: Perform a Legitimate Reset on Your Own Account

With Burp running, click **Forgot your password?** and enter your own username. Open the **Email client**, follow the reset link from the email, and set a new password of your choice.

### Step 2: Study the Reset Request in Burp

In **Proxy > HTTP history**, review the reset traffic. Two observations matter:

- The reset token travels as a URL query parameter named `temp-forgot-password-token`.
- The final `POST /forgot-password?temp-forgot-password-token=...` request that submits the new password also carries the **username as a hidden input** in the request body.

Send this POST request to Burp Repeater.

### Step 3: Prove the Token Is Not Validated

In Repeater, delete the **value** of the `temp-forgot-password-token` parameter — in both the URL and the request body — and resend. The password reset still works. This confirms the token is never checked on submission; it only decorates the email link.

### Step 4: Repeat the Request, This Time Targeting Carlos

Back in the browser, request another password reset for your own account and change your password again, then send the new `POST /forgot-password` request to Repeater once more.

### Step 5: Swap the Username and Reset Carlos's Password

In Repeater:

1. Delete the `temp-forgot-password-token` value in both the URL and the body (as proven in Step 3, it is not needed).
2. Change the `username` parameter to `carlos`.
3. Set the new password fields to a password of your choosing.
4. Send the request.

The server resets Carlos's password to your chosen value — no token, no email access, no proof of identity.

### Step 6: Log In as Carlos

In the browser, log in to Carlos's account with the new password you just set and open **My account** to solve the lab.

## Reference Screenshot

<img width="1995" height="1569" alt="step-1" src="https://github.com/user-attachments/assets/a17dc8d0-adbf-4c15-9c50-384b59b4c056" />
## Key Takeaways

- A reset token provides security only if the server verifies it — and binds it to the account — at the moment the new password is submitted. A token that is checked nowhere is pure decoration.
- Identifying the target account from a client-supplied `username` field in the reset request turns a broken check into full account takeover of any user.
- Always test reset flows by removing the token, replaying requests, and swapping the username; these three quick checks expose most broken reset logic.

## Remediation

- Generate reset tokens that are long, random, single-use, and short-lived; store them server-side, bound to the account that requested the reset.
- On password submission, require a valid token and derive the target account **from the token**, never from a client-supplied username.
- Invalidate the token immediately after use and notify the account owner of the change via email.
