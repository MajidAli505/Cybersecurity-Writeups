# Authentication Bypass via Information Disclosure

**Platform:** PortSwigger Web Security Academy  
**Category:** Information Disclosure  
**Difficulty:** Apprentice  
**Status:** ✅ Solved

## Overview

Some applications sit behind a front-end proxy that vouches for where a request came from. The proxy stamps each incoming request with a custom header recording the client's IP address, and the back-end treats requests as "local" — and therefore trusted — when that header contains a localhost address. The design fails twice over. First, the back-end trusts a header value it cannot verify. Second, the header's very existence can be discovered: the HTTP `TRACE` method makes the server echo back the request exactly as it received it, *including* headers the proxy added along the way. That echo is the information disclosure; spoofing the header is the authentication bypass.

## Lab Objective

Discover the custom header the front-end uses, spoof it to make your requests appear to come from localhost, access the admin panel, and delete the user `carlos`.

**Provided resources:** your credentials `wiener:peter`.

<img width="2183" height="1032" alt="step-1" src="https://github.com/user-attachments/assets/a7ea877c-0a2b-4208-8d04-c70a086349e8" />

<img width="3153" height="1440" alt="step-2" src="https://github.com/user-attachments/assets/243122d3-e73f-425f-89fe-10a19ac6bdec" />

## Walkthrough

### Step 1: Probe the Admin Panel

In Burp Repeater, request `GET /admin`. The response explains the access rule in plain language: the admin panel is available only to logged-in administrators, **or to requests originating from a local (localhost) IP address**. That second condition is the target — the question is how the application decides a request is local.

<img width="2725" height="1348" alt="step-3" src="https://github.com/user-attachments/assets/907d2e28-6904-4fc1-a027-5a0bc425cda3" />

<img width="1571" height="762" alt="step-4" src="https://github.com/user-attachments/assets/ac4abdc6-78d8-4062-94ec-d9ed7733be13" />

### Step 2: Reveal the Hidden Header with TRACE

Send the same request again, but switch the method to `TRACE`:

```http
TRACE /admin
```

A `TRACE` response echoes the request as the server received it. Comparing the echo with what was sent reveals a header that was never added by the browser: the front-end proxy has appended

```http
X-Custom-IP-Authorization: <your real IP address>
```

This header is clearly how the back-end identifies the client's origin — and therefore what "local" means in practice.

<img width="2768" height="1327" alt="step-5" src="https://github.com/user-attachments/assets/50ea4190-0ade-4408-bf2f-8141c9063bbe" />

### Step 3: Configure Burp Match and Replace

Open **Proxy → Match and replace** and add a new HTTP rule:

- **Type:** Request header
- **Match:** *(leave empty)*
- **Replace:** `X-Custom-IP-Authorization: 127.0.0.1`

Testing the rule shows Burp automatically attaching the header, with the localhost value, to the modified request. With an empty match field, the rule applies to every request.

<img width="2398" height="900" alt="step-6" src="https://github.com/user-attachments/assets/3e099aaf-e74e-4892-9665-0cabcd9f1f82" />

<img width="3051" height="1707" alt="step-7" src="https://github.com/user-attachments/assets/b7923af9-7620-48ce-9cd8-12a69020ce10" />

<img width="3145" height="1033" alt="step-8" src="https://github.com/user-attachments/assets/3867919a-790b-4dd0-9f38-28e02b1c3b1a" />

### Step 4: Access the Admin Panel and Delete Carlos

Click **OK** to enable the rule, then simply browse the site. Because every request now carries `X-Custom-IP-Authorization: 127.0.0.1`, the application treats you as a local user and the **Admin panel** link becomes available. Open the panel and delete the user `carlos`.

<img width="3042" height="824" alt="step-9" src="https://github.com/user-attachments/assets/3b0a3c13-a290-4ec1-a487-db63ef18cc9e" />

### Step 5: Lab Solved

The panel confirms "**User deleted successfully!**" and the lab is complete — reached not by stealing credentials, but by learning one header name and lying about an IP address.

<img width="3061" height="1102" alt="step-10" src="https://github.com/user-attachments/assets/626ba0eb-0c21-4d3b-ad1e-290b23ceadbb" />

## Key Takeaways

- `TRACE` is a diagnostic method that echoes requests back, exposing headers added by intermediaries. It should be disabled in production; its reflection behavior (cross-site tracing) is a disclosure channel.
- Trust decisions must never rest on request headers an attacker can set. If a client can supply or override the header, the "trusted origin" check is decorative.
- Error and access-denied messages that explain the exact access rule ("admins or localhost") hand attackers a roadmap of what to spoof.

## Remediation

- Disable the `TRACE` method on servers and proxies.
- Strip any client-supplied internal headers (such as custom IP-authorization headers) at the edge, and set them only from values the proxy itself verified.
- Base administrative access on authenticated identity and authorization checks in the application, not on apparent network origin; keep denial messages generic.
