# Bypassing SameSite Restrictions Using On-Site Gadgets

**Platform:** PortSwigger Web Security Academy  
**Lab:** SameSite Strict bypass via client-side redirect  
**Category:** Cross-Site Request Forgery (CSRF)  
**Difficulty:** Practitioner  
**Status:** ✅ Solved

## Overview

This lab demonstrates how to bypass `SameSite=Strict` cookie restrictions by abusing a legitimate feature of the target website itself — an **on-site gadget**.

The application's change-email function is vulnerable to CSRF. It contains no unpredictable anti-CSRF token, and the application relies on `SameSite=Strict` session cookies to prevent cross-site requests. That protection can be bypassed completely if the site contains a client-side redirect that builds its target from attacker-controlled input.

## Objective

To solve the lab, perform a CSRF attack that changes the victim's email address, using the provided exploit server to host the attack.

## Background: What Does `SameSite=Strict` Do?

Suppose a user is logged in to:

```text
https://bank.com
```

The browser holds their session cookie:

```text
session=abc123
SameSite=Strict
```

Now the user visits an attacker-controlled site:

```text
https://evil.com
```

If `evil.com` tries to make the victim's browser send a request to `https://bank.com/transfer`, the browser classifies it as a **cross-site** request and, because of `SameSite=Strict`, does **not** send the cookie.

Without the session cookie, the request is unauthenticated, so the CSRF attack fails. This is why `SameSite=Strict` is normally an effective defence.

## The Problem: On-Site Gadgets

Some websites contain JavaScript that performs redirects based on user-controlled input, for example:

```javascript
let url = new URLSearchParams(location.search).get("next");
location = url;
```

A page such as `https://bank.com/redirect?next=/profile` reads the `next` parameter and immediately redirects the browser to that path. The redirect target is therefore controlled by whoever supplies the link.

### Why This Is Dangerous

The key insight is that **the site changes during the redirect**:

1. The victim's browser makes the first request cross-site (`evil.com` → `bank.com`). The cookie is correctly blocked on this request.
2. The page that loads is now running on `bank.com`, and its JavaScript initiates a second request.
3. From the browser's perspective, this second request is `bank.com` → `bank.com` — a **same-site** request.
4. `SameSite` restrictions only apply to cross-site requests, so the session cookie **is** sent with the second request.

```text
Normal blocked attack                 Bypass using a gadget

evil.com                              evil.com
  |                                     |
  | POST /change-email                  | Open bank.com/redirect?next=/change-email
  v                                     v
bank.com                              bank.com/redirect page
                                        |
SameSite=Strict                         | JavaScript redirect
Cookie blocked                          v
                                      bank.com/change-email
Attack fails
                                        Cookie included (same-site request)
                                        Attack succeeds
```

A **gadget** is a piece of legitimate website functionality that an attacker can abuse for another purpose. Examples include open redirects, client-side redirects, dynamic URL loading, DOM-based navigation, auto-submitting forms, and AJAX requests built from URL parameters. None of these is necessarily a vulnerability on its own, but combined with `SameSite` behaviour they can become a complete bypass.

## Walkthrough

### Step 1: Study the Change-Email Function

I logged in to my own account in Burp's browser and changed my email address. In **Proxy > HTTP history**, I studied the resulting request:

```text
POST /my-account/change-email
```

It contained no unpredictable tokens, so it was potentially vulnerable to CSRF — if the `SameSite` restrictions could be bypassed.

In the response to the `POST /login` request, the application explicitly set its session cookies with `SameSite=Strict`, confirming that the browser would not include them in cross-site requests.

### Step 2: Submit a Blog Comment

Next, I submitted a comment on a blog post. After submitting, I was first taken to a confirmation page:

```text
/post/comment/confirmation?postId=1
```

A few seconds later, without clicking anything, the browser automatically returned to the blog post itself:

```text
/post/1
```

That raised the obvious question: was this redirect performed by the server, or by JavaScript?

### Step 3: Check the HTTP History in Burp

In Burp's HTTP history, I found that the confirmation page had loaded a JavaScript file:

```text
/resources/js/commentConfirmationRedirect.js
```

This file became the main point of interest.

### Step 4: Examine the JavaScript

The script constructed the redirect target dynamically from the `postId` query parameter, along these lines:

```javascript
var postId = new URLSearchParams(location.search).get('postId');
location = "/post/" + postId;
```

### Step 5: The Gadget

This was the gadget. The developer used the user-controlled `postId` parameter directly to build the redirect URL:

```javascript
location = "/post/" + postId;
```

I confirmed that the redirect destination was fully controllable:

| Request | Redirected to |
|---|---|
| `/post/comment/confirmation?postId=1` | `/post/1` |
| `/post/comment/confirmation?postId=foo` | `/post/foo` |

### Step 6: Escape the `/post/` Prefix with Path Traversal

Because `/post/` was automatically prepended to my input, I used a path traversal sequence to escape it:

```text
/post/comment/confirmation?postId=1/../../my-account
```

The JavaScript built the following path:

```text
/post/1/../../my-account
```

The browser normalized this URL to:

```text
/my-account
```

I was successfully redirected to my account page. This confirmed that the `postId` parameter could be used to make the browser issue a `GET` request to an arbitrary endpoint on the target site — and that this request, being same-site, would carry the session cookie despite the attack starting from an external site.

### Step 7: Confirm the SameSite Bypass

On the exploit server, I hosted a script that sent the viewer to the gadget URL:

```html
<script>
    document.location = "https://YOUR-LAB-ID.web-security-academy.net/post/comment/confirmation?postId=../my-account";
</script>
```

When I stored and viewed the exploit myself, the client-side redirect still landed me on my logged-in account page. This confirmed that the browser included my authenticated session cookie in the **second** request — the one generated by the gadget — even though the initial request was initiated from an external site.

> The cookie was not sent with the first request. It was sent with the second request, generated by the on-site gadget. That is the entire trick behind this `SameSite=Strict` bypass.

### Step 8: Craft the Final Exploit

1. I sent the `POST /my-account/change-email` request to Burp Repeater.
2. Using **Change request method**, Burp generated an equivalent `GET` request, and sending it confirmed that the endpoint also accepted email changes via `GET`.
3. I updated the exploit so the gadget's redirect would trigger that `GET` change-email request:

```html
<script>
    document.location = "https://YOUR-LAB-ID.web-security-academy.net/post/comment/confirmation?postId=1/../../my-account/change-email?email=pwned%40web-security-academy.net%26submit=1";
</script>
```

Note that the `submit` parameter had to be included, and the ampersand (`&`) had to be URL-encoded as `%26` so it would not break out of the `postId` parameter in the initial request.

4. I tested the exploit against myself and confirmed my email address was changed.
5. I changed the email address in the exploit to one that did not match my own account and delivered the exploit to the victim. After a few seconds, the lab was solved.

## Attack Chain Summary

```text
Victim visits the exploit page
        ↓
Request 1: evil.com → victim.com/post/comment/confirmation
Cross-site — SameSite=Strict cookie blocked
        ↓
Client-side JavaScript gadget executes on victim.com
        ↓
Request 2: victim.com → victim.com/my-account/change-email
Same-site — session cookie sent
        ↓
Victim's email address is changed
```

## Key Takeaways

- `SameSite=Strict` blocks cookies on cross-site requests, but it does not apply once a request is initiated from within the site itself.
- A client-side redirect that builds its target from user input is an on-site gadget: it converts a cross-site attack into a same-site request that carries cookies.
- A gadget is rarely dangerous in isolation. It becomes critical when combined with a sensitive, token-less state-changing endpoint — especially one that also accepts `GET` requests.

## Remediation

- Protect state-changing endpoints with unpredictable anti-CSRF tokens; do not rely on `SameSite` cookies as the only defence.
- Do not allow state-changing actions via `GET` requests.
- Avoid building redirect targets directly from unvalidated user input. Where redirects based on parameters are required, validate the target against an allowlist of local paths.
- Audit the whole site — including on-site gadgets such as client-side redirects — when assessing CSRF defences, because a single gadget can undermine them completely.
