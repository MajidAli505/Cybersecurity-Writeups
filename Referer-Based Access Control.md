# Referer-Based Access Control

**Platform:** PortSwigger Web Security Academy
**Category:** Access Control
**Difficulty:** Practitioner
**Status:** ✅ Solved

## Background

When a browser sends a request from one page to another, it usually includes a `Referer` header stating which page the request came from:

```http
GET /admin/deleteUser?id=5 HTTP/1.1
Host: example.com
Referer: https://example.com/admin
```

From that header, the server can see the request originated on the `/admin` page. The vulnerability appears when a developer turns that observation into an access control:

- If `Referer` is `/admin` → allow the request.
- Otherwise → reject it.

What the server is *not* checking is the only thing that matters: whether the user is an admin, whether they are even authenticated, or whether they hold the permission at all. The `Referer` header is supplied by the client, so anyone can set it to anything — in Burp Suite, it is just another line to type. A control based on a value the attacker writes is not a control.

## Lab Objective

This lab controls access to certain admin functionality based on the `Referer` header. Log in as `wiener:peter` and exploit the flawed access controls to promote yourself to become an administrator.

**Provided resources:** you can familiarize yourself with the admin panel first using the credentials `administrator:admin`.
<img width="2178" height="926" alt="step-1" src="https://github.com/user-attachments/assets/d52073cb-00fc-40aa-a8c0-15426aa76714" />


## Walkthrough

### Step 1: Perform a Genuine Upgrade as Admin

Log in using the admin credentials and go to the admin panel. Click "Upgrade user" for `carlos` — he becomes an admin. More importantly, capture the upgrade request in Burp; this is the request the server trusts:

```http
GET /admin-roles?username=carlos&action=upgrade HTTP/2
Host: <lab host>
Cookie: session=<administrator's session>
Referer: https://<lab host>/admin
```

Notice the `Referer` header pointing back at `/admin` — that, not the powerful session, is what this endpoint actually relies on.
<img width="3274" height="1821" alt="step-2" src="https://github.com/user-attachments/assets/91bbd524-8333-4a6c-a15d-fd5abe7aa729" />


<img width="3055" height="1398" alt="step-3" src="https://github.com/user-attachments/assets/2fb44d0f-8e62-4e07-897d-31c7c774ba09" />


<img width="3064" height="1404" alt="step-4" src="https://github.com/user-attachments/assets/ad0056b3-85ae-4a05-928f-4a44fc705c63" />


<img width="3003" height="932" alt="step-5" src="https://github.com/user-attachments/assets/b922128c-5075-4893-abc9-9327244d2e9e" />


<img width="3126" height="1032" alt="step-6" src="https://github.com/user-attachments/assets/8fc16094-8618-4b76-9f48-c04c0655dfd4" />


<img width="2686" height="1466" alt="step-7" src="https://github.com/user-attachments/assets/b1b60883-78ce-40e6-8219-bc5a6a0f542c" />


### Step 2: Get a Non-Admin Session

Now log in with the non-administrator account (`wiener:peter`) and copy that user's session cookie.

<img width="2449" height="1415" alt="step-8" src="https://github.com/user-attachments/assets/ced0230f-2990-4a24-a76e-c28ec105a090" />


### Step 3: Replay the Request with the Forged Context

Modify the captured admin request: paste in the non-administrator's session cookie, and change the username to `wiener`. Leave everything else — above all the `Referer: .../admin` header — untouched, and send it:

```http
GET /admin-roles?username=wiener&action=upgrade HTTP/2
Host: <lab host>
Cookie: session=<wiener's session>
Referer: https://<lab host>/admin
```

The server sees a request that "came from" the admin page and applies the upgrade. At no point does it verify that the session behind the request belongs to an administrator.

<img width="2786" height="1756" alt="step-9" src="https://github.com/user-attachments/assets/a622529f-0d48-44dc-9484-c92d9c705d67" />


<img width="2392" height="1283" alt="step-10" src="https://github.com/user-attachments/assets/4ca09907-bf0c-4853-a422-e4e23699d93b" />


<img width="2621" height="1296" alt="step-11" src="https://github.com/user-attachments/assets/73f26625-1f54-4bb7-93a3-e7b2766236fb" />


<img width="2567" height="1164" alt="step-12" src="https://github.com/user-attachments/assets/98f28e03-5f1e-45a2-91b7-0c06681d76c0" />


### Step 4: Confirm the Promotion

`wiener` is now an administrator — the admin panel is available to his account, and the lab is solved.

<img width="3121" height="1027" alt="step-13" src="https://github.com/user-attachments/assets/ed9b3dd2-94e9-46fb-a2b6-4247cc9d7a33" />


## Key Takeaways

- The `Referer` header describes where a request claims to come from; claims made by the client are evidence of nothing. Like every header, it is fully attacker-controlled.
- Guessing the check was easy because the genuine request shows it: any header a legitimate request carries can be copied into a forged one.
- This flaw recurs in many costumes — trusting `X-Forwarded-For`, an origin header, or a hidden form field. The pattern to recognize is an authorization decision based on request metadata instead of the authenticated identity.

## Remediation

Base every authorization decision on the authenticated session and the user's actual permissions, evaluated server-side for each protected action. Headers such as `Referer` may be logged or used for analytics, but they must never gate functionality. (Referer checks have a legitimate, narrow use as a defense-in-depth layer against CSRF — never as the access control itself.)
