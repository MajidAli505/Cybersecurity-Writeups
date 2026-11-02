# Method-Based Access Control Can Be Circumvented

**Platform:** PortSwigger Web Security Academy
**Category:** Access Control
**Difficulty:** Practitioner
**Status:** ✅ Solved

## Overview

Some applications bolt their authorization checks onto specific HTTP methods — typically verifying "is this user an admin?" only on `POST` requests, because that is the method the interface uses for sensitive actions. The endpoint that performs the action, however, often accepts other methods too. If the check and the action are keyed to different properties of the request, an attacker can keep the action and shed the check simply by changing the method. This lab's upgrade endpoint is guarded exactly that way, and the giveaway is that different methods produce different errors.

## Lab Objective

This lab implements access controls based partly on the HTTP method of requests. Log in as `wiener:peter` and exploit the flawed access controls to promote yourself to become an administrator.

**Provided resources:** you can familiarize yourself with the admin panel first using the credentials `administrator:admin`.

<img width="2217" height="1035" alt="step-1" src="https://github.com/user-attachments/assets/9fdf880e-6b3f-42ae-bb44-6c89fb5f9c49" />

## Walkthrough

### Step 1: Study the Genuine Admin Request

Log in using the admin credentials. Browse to the admin panel, promote `carlos`, and send the HTTP request to Burp Repeater. This gives you the exact shape of a legitimate upgrade request — a `POST` to the admin upgrade endpoint carrying the target username — performed with an admin session.

<img width="3289" height="1904" alt="step-2" src="https://github.com/user-attachments/assets/f1479eb5-f39b-4227-87a2-6ced422fbca4" />

<img width="2545" height="1373" alt="step-3" src="https://github.com/user-attachments/assets/e75f445e-777b-43f2-a762-821b0db1fc0a" />

<img width="3105" height="940" alt="step-4" src="https://github.com/user-attachments/assets/052afbb1-9e29-44b8-b6a6-e8799fd07e75" />

<img width="2653" height="1274" alt="step-5" src="https://github.com/user-attachments/assets/25837d95-a244-4370-b5a6-29a275ff63d1" />

### Step 2: Replay It as a Normal User

Open a private/incognito browser window, and log in with the non-admin credentials (`wiener:peter`). Attempt to re-promote `carlos` with the non-admin user by copying that user's session cookie into the existing Burp Repeater request, and observe that the response says "Unauthorized". The control works — for this exact request shape.

<img width="3662" height="1675" alt="step-6" src="https://github.com/user-attachments/assets/77ebaca4-ca37-4f55-95c3-ee27af4e1d03" />

<img width="3448" height="1633" alt="step-7" src="https://github.com/user-attachments/assets/588a5cf7-41f0-486f-a20a-12cc8047f1d1" />

<img width="2788" height="1330" alt="step-8" src="https://github.com/user-attachments/assets/3997c194-3b49-4469-ac43-c05edc1a04b7" />

### Step 3: Change the Method and Watch the Error Change

Change the method from `POST` to `POSTX` and observe that the response changes to "missing parameter". The bogus method slipped straight past the authorization check — the request now fails only because the back-end handler expects its parameters in a different place for non-POST methods. A different error means a different code path, and this one has no admin check on it.

<img width="2436" height="1341" alt="step-9" src="https://github.com/user-attachments/assets/f4223176-c0f7-4430-8a4e-9b144edec41f" />

### Step 4: Convert to GET and Promote Yourself

Convert the request to use the `GET` method by right-clicking and selecting "Change request method", which moves the parameters into the query string where this code path looks for them. Change the username parameter to your own username and resend the request:

```http
GET /admin-roles?username=wiener&action=upgrade HTTP/2
Host: <lab host>
Cookie: session=<wiener's session>
```

The request succeeds. Log in again (or refresh) as `wiener` and the admin panel is now available to you — you have promoted yourself to administrator, and the lab is solved.

<img width="2807" height="1502" alt="step-10" src="https://github.com/user-attachments/assets/c82e32d0-36ae-4ff6-8213-d8f02e5b2320" />

<img width="2344" height="1151" alt="step-11" src="https://github.com/user-attachments/assets/f17d15a6-2d94-4905-aaaa-b015be2916e2" />

<img width="2972" height="1624" alt="step-12" src="https://github.com/user-attachments/assets/f86e0557-df04-4d3e-b456-70bef3d858ef" />

<img width="3290" height="1163" alt="step-13" src="https://github.com/user-attachments/assets/01d8deb9-b4d1-4e02-b253-40cba066595d" />

## Key Takeaways

- An authorization check that only runs for one HTTP method protects that method, not the action. Any other method the endpoint accepts is an unguarded door to the same functionality.
- Changed error messages are a detection signal: "Unauthorized" becoming "missing parameter" proved the request had escaped the checked code path before any exploit was even completed.
- Learning the legitimate request first (with the provided admin account) turned the attack into simple differential testing: same request, one variable at a time — session, then method.

## Remediation

Apply authorization checks per action, not per method: every request that reaches a sensitive function should pass the same check, whatever its method. Ideally the check lives in one place — middleware or a decorator on the endpoint itself — so no route or verb can bypass it. Endpoints should also reject unexpected methods outright rather than falling through to alternate handling.
