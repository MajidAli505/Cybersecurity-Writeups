# Bypassing Referer-Based CSRF Defenses

**Platform:** PortSwigger Web Security Academy  
**Category:** Cross-Site Request Forgery (CSRF)  
**Status:** ✅ Solved

## Overview

This lab's email-change functionality is vulnerable to CSRF. The application attempts to block cross-domain requests by validating the `Referer` header, but it has an **insecure fallback**: if the `Referer` header is missing entirely, the request is accepted.

That single flaw is enough to bypass the defence completely, because an attacker page can instruct the browser not to send a `Referer` header at all.

## Objective

To solve the lab, use the exploit server to host an HTML page that performs a CSRF attack and changes the victim's email address.

The lab account credentials are `wiener:peter`.

## Background: Referer-Based CSRF Validation

Some applications try to prevent CSRF by checking the `Referer` header of sensitive requests:

- If the `Referer` contains the application's own domain, the request is treated as legitimate.
- If the `Referer` contains a different domain, the request is rejected as a cross-site forgery attempt.

This defence depends on one assumption — that the header will always be present. Applications often fail **open** when it is absent, for example to avoid breaking requests that legitimately arrive without a `Referer`. An attacker can exploit exactly that gap by suppressing the header.

## Walkthrough

### Step 1: Capture the Change-Email Request

I logged in to my account in Burp's browser using the credentials `wiener:peter` and submitted the **Update email** form. I then located the resulting request in the Proxy history.

### Step 2: Confirm That Referer Validation Exists

I sent the request to Burp Repeater and changed the domain in the `Referer` header to a different domain. The request was **rejected**, confirming that the application was validating the `Referer` header for CSRF protection.

### Step 3: Find the Insecure Fallback

Next, I deleted the `Referer` header from the request entirely and sent it again. This time the request was **accepted**, and the email change went through.

This confirmed the vulnerability:

| Request | Result |
|---|---|
| `Referer` with a different domain | ❌ Rejected |
| No `Referer` header at all | ✅ Accepted |

The application validated the header only when it was present, and allowed the request when it was missing.

### Step 4: Build the Exploit

I created a CSRF proof of concept on the exploit server: an auto-submitting form that changes the viewer's email address, of the form:

```html
<form action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email" method="POST">
    <input type="hidden" name="email" value="pwned@web-security-academy.net">
</form>
<script>
    document.forms[0].submit();
</script>
```

The key addition was a single line in the page head that instructs the browser to suppress the `Referer` header for requests originating from the page:

```html
<meta name="referrer" content="no-referrer">
```

With this in place, the victim's browser submitted the form **without** any `Referer` header, hit the application's insecure fallback, and the request was accepted.

### Step 5: Deliver the Exploit

I changed the email address in the exploit so it did not match my own account, stored the exploit, and clicked **Deliver to victim**. The victim's browser executed the page, their email address was changed, and the lab was solved.

## Key Takeaways

- Validating the `Referer` header is only a defence if the application also handles its **absence** securely. Failing open when the header is missing makes the validation trivially bypassable.
- The `referrer` meta tag (`<meta name="referrer" content="no-referrer">`) gives any web page a standard, reliable way to suppress the `Referer` header — no browser bug or special tooling is required.

## Remediation

- Do not use `Referer` validation as the primary CSRF defence. If it is used at all, requests with a missing `Referer` header must be **rejected**, not accepted.
- Protect all state-changing endpoints with unpredictable anti-CSRF tokens, which do not depend on browser header behaviour.
