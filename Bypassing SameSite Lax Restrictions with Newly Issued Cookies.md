# Bypassing SameSite Lax Restrictions with Newly Issued Cookies

**Platform:** PortSwigger Web Security Academy  
**Category:** Cross-Site Request Forgery (CSRF)  
**Status:** ✅ Solved

## Overview

This lab demonstrates how to bypass `SameSite=Lax` cookie restrictions by exploiting Chrome's temporary exception for **newly issued cookies**.

Normally, `SameSite=Lax` prevents a session cookie from being sent with a cross-site `POST` request, which blocks CSRF. However, if a cookie was created very recently and the application did not explicitly set a `SameSite` attribute, Chrome relaxes that restriction for the first **120 seconds (2 minutes)**. An attacker who can force the victim's browser to obtain a fresh session cookie — for example through an OAuth login flow — can launch a CSRF attack inside that window.

## Background: Normal `SameSite=Lax` Behaviour

Suppose a user is logged in to `https://bank.com` and the browser stores their session cookie as `SameSite=Lax`. If an attacker site, `https://evil.com`, auto-submits a form to the target:

```html
<form action="https://bank.com/transfer" method="POST">
    <input type="hidden" name="amount" value="1000">
</form>
<script>
    document.forms[0].submit();
</script>
```

Because this is a cross-site `POST` request, the browser does **not** send the Lax cookie. The request arrives unauthenticated, and the CSRF attack fails.

## The Chrome Exception

Chrome applies a special exception to cookies that meet two conditions:

1. The cookie is **newly created**.
2. The server **did not explicitly specify** a `SameSite` attribute, so Chrome applies `SameSite=Lax` by default.

For the first 120 seconds after such a cookie is created, Chrome temporarily allows it to be sent with a top-level cross-site `POST` request. This grace period was introduced so that older single sign-on (SSO) systems would not break.

### Example

1. **The victim logs in.** The server responds with a cookie and no explicit `SameSite` attribute:

   ```text
   Set-Cookie: session=abc123
   ```

   Chrome treats it as `Lax` by default and starts the 120-second timer.

2. **The attacker triggers a POST request within those 120 seconds**, using an auto-submitted form like the one above.

Because the cookie is still very new, Chrome sends it with the request. The server sees a valid session, processes the request, and the CSRF attack succeeds.

## Why This Matters

Many websites rely solely on `SameSite=Lax` for CSRF protection. The attack becomes possible when all of the following are true:

1. The application issues a fresh session cookie.
2. It does not explicitly set `SameSite=Lax` on that cookie.
3. It has a sensitive `POST` endpoint.
4. It does not use anti-CSRF tokens.

In that case, the 2-minute window is enough to exploit the endpoint.

## Explicit vs. Implicit Lax

| Cookie | Chrome's behaviour |
|---|---|
| **Implicit Lax** — `Set-Cookie: session=abc123` (no `SameSite` attribute) | Chrome assumes `SameSite=Lax`, **but** applies the 120-second grace period. Vulnerable to this bypass. |
| **Explicit Lax** — `Set-Cookie: session=abc123; SameSite=Lax` | Chrome enforces Lax immediately, with **no** exception. Cross-site `POST` requests do not carry the cookie, and CSRF is blocked. |

## The Attacker's Problem

The attacker cannot predict when the victim will log in, so simply waiting for a naturally fresh cookie is unreliable. The challenge is therefore:

> How can the attacker **force** a new session cookie to be issued, exactly when they need it?

## Solution: Force a New Session Cookie via OAuth

The most common way to force a fresh cookie is to find a feature that issues a new session cookie every time it is used. An **OAuth login flow** ("Login with Google, Facebook, GitHub", etc.) is the classic example.

### Attack Flow

**Step 1 — The victim is already logged in** to the target site, with an existing (old) session cookie.

**Step 2 — The victim visits the attacker's site.**

**Step 3 — The attacker redirects the victim into the target's OAuth login flow.** The victim does not need to type any credentials: their identity-provider cookies are already in the browser, so OAuth authenticates them automatically.

**Step 4 — The OAuth flow completes and the target issues a new session cookie:**

```text
Set-Cookie: session=NEW456
```

Chrome's 120-second grace period starts at this exact moment.

**Step 5 — The victim must be brought back to the attacker's site.** This can be achieved through the OAuth flow's `redirect_uri` or through an open redirect:

```text
evil.com → bank.com OAuth → Identity provider → bank.com → evil.com
```

**Step 6 — The attacker's page immediately auto-submits the CSRF form.** The browser sends the cross-site `POST`, and because the session cookie was issued only moments earlier, Chrome includes it. The CSRF attack succeeds.

### Why Top-Level Navigation Is Required

The OAuth cookies are only sent correctly if the victim performs an actual page navigation — a **top-level navigation** — such as `window.location`, a `302` redirect, or clicking a link. If the attacker uses only an `<iframe>` or a background request, the OAuth cookies may not be sent properly and the login flow can fail.

### Summary

- **Goal:** Obtain a fresh session cookie for the victim.
- **Method:** Trigger the OAuth login flow.
- **Reason:** Chrome relaxes the Lax restriction for 120 seconds on a newly issued cookie.
- **Challenge:** Bring the victim back to the attacker site after the OAuth flow completes.

## A Better Approach: Open the OAuth Flow in a New Tab

Instead of redirecting the victim away, the attacker can keep the malicious page open and open the OAuth flow in a separate tab:

```javascript
window.open('https://bank.com/login/sso');
```

The result:

```text
Tab 1: evil.com         (stays open)
Tab 2: bank.com OAuth   (completes, fresh cookie issued)
```

Once OAuth completes, the new session cookie is stored in the browser's cookie jar, and the victim is still sitting on the attacker's page, ready for the CSRF request.

### Workaround for Popup Blockers

Browsers block `window.open()` calls that run automatically on page load, treating them as unwanted popups. A popup **is** allowed, however, when it follows a user action such as a click, a button press, or a link click:

```javascript
window.onclick = () => {
    window.open('https://bank.com/login/sso');
};
```

### Why This Works

Cookies are stored at the **browser level**, not the tab level. The cookie created in the OAuth tab is therefore immediately available to requests initiated from the attacker's tab:

```text
Victim clicks on evil.com
        ↓
OAuth flow opens in a new tab
        ↓
Fresh session cookie is issued and stored in the browser
        ↓
Original evil.com page remains open
        ↓
CSRF attack is launched within 120 seconds, cookie attached
```

In short, this lab is a classic chain:

```text
OAuth login  +  Newly issued cookie  +  Lax 120-second exception  +  CSRF
```

Together, they bypass `SameSite=Lax` protection whenever the cookie does not explicitly set `SameSite=Lax`.

## Key Takeaways

- `SameSite=Lax` by default is not the same as `SameSite=Lax` set explicitly: only the implicit version receives Chrome's 120-second grace period for newly issued cookies.
- The grace period exists for SSO compatibility, but any flow that reliably re-issues a session cookie (especially OAuth) gives an attacker control over when that window opens.
- Timing is the whole attack: force the fresh cookie, return the victim to the attacker page, and fire the CSRF request inside two minutes.

## Remediation

- Set the `SameSite` attribute **explicitly** on session cookies (for example, `SameSite=Lax` or `SameSite=Strict`) so no implicit-Lax grace period applies.
- Do not rely on `SameSite` alone: use unpredictable anti-CSRF tokens on all state-changing requests.
- Re-issuing session cookies after authentication is good practice, but it must be combined with the controls above so a fresh cookie cannot be abused.
