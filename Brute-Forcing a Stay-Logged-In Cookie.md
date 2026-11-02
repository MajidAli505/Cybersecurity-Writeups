# Brute-Forcing a Stay-Logged-In Cookie

**Platform:** PortSwigger Web Security Academy  
**Category:** Authentication Vulnerabilities  
**Difficulty:** Practitioner  
**Status:** ✅ Solved

## Overview

This lab's "stay logged in" feature is implemented with a persistent cookie that is vulnerable to brute-forcing. The cookie is not a random token — it is constructed deterministically from the user's credentials as:

```text
stay-logged-in = Base64( username : MD5(password) )
```

Because the format is predictable and a candidate password list is provided, an attacker can forge a valid cookie for the victim, `carlos`, by generating the cookie value for each candidate password and testing them all with Burp Intruder.

## Lab Objective

Brute-force Carlos's stay-logged-in cookie to gain access to his **My account** page.

**Provided resources:** your credentials `wiener:peter`, the victim's username `carlos`, and a list of candidate passwords.

## Walkthrough

### Step 1: Review the Lab Brief

The lab page confirms the target: the stay-logged-in cookie is vulnerable to brute-forcing, and solving the lab means accessing Carlos's account page.

<img width="2184" height="920" alt="step-1" src="https://github.com/user-attachments/assets/34d4bfe2-415e-4317-8c9c-a2e744d7cf65" />

### Step 2: Log In with "Stay Logged In" Enabled

Log in with your own credentials (`wiener:peter`), making sure the **Stay logged in** checkbox is ticked so the application issues the persistent cookie.

<img width="3150" height="1296" alt="step-2" src="https://github.com/user-attachments/assets/5d46bbce-a116-4f0f-ad84-1440633df8a5" />

### Step 3: Locate the Cookie in Burp

Browse to your account page and find the `GET /my-account?id=wiener` request in Burp's HTTP history. The request carries a `stay-logged-in` cookie with a long Base64-looking value.

<img width="2890" height="1371" alt="step-3" src="https://github.com/user-attachments/assets/7c0c8288-13b6-4221-9ad4-8a3db75fcde6" />

### Step 4: Decode the Cookie Format

Send the cookie value to Burp Decoder and decode it as **Base64**. The decoded value reveals the cookie's internal structure:

```text
wiener:51dc30ddc473d43a6011e9ebba6ca770
```

That is `username:hash` — and the hash is 32 hex characters, the signature length of MD5.

<img width="3837" height="887" alt="step-4" src="https://github.com/user-attachments/assets/c3f3f70f-d553-41f2-add1-b1e18f721e81" />

<img width="1499" height="536" alt="step-5" src="https://github.com/user-attachments/assets/c0078653-0601-44f9-b258-5bebda3fac5a" />

### Step 5: Confirm the Hash Is the MD5 of the Password

Checking the hash against CrackStation confirms it: the value is the **MD5** hash of `peter` — my own account password. The cookie format is now proven:

```text
Base64( username : MD5(password) )
```

So for Carlos, each candidate cookie must be generated as `Base64(carlos:MD5(candidate))`.

<img width="2649" height="1001" alt="step-6" src="https://github.com/user-attachments/assets/a3f2755f-b267-4f5a-ba3c-f7b020a02afc" />

### Step 6: Send the Account Request to Intruder

Send the `GET /my-account` request to Burp Intruder (Sniper attack). Remove the `session` cookie from the request, so that authentication depends only on the `stay-logged-in` cookie being tested.

<img width="3054" height="1009" alt="step-7" src="https://github.com/user-attachments/assets/97dd2191-7b6d-4936-aa05-2ab5c96ff730" />

<img width="2706" height="1348" alt="step-8" src="https://github.com/user-attachments/assets/678dca0c-18db-489d-be2c-33fd70a2507c" />

<img width="2864" height="1454" alt="step-9" src="https://github.com/user-attachments/assets/4afbe818-ee9c-4a05-859e-c05158a9bf38" />

### Step 7: Configure the Payload Position and Password List

Place the payload position over the **entire** `stay-logged-in` cookie value, and load the candidate password list (100 passwords) as the payload set.

<img width="2099" height="1686" alt="step-10" src="https://github.com/user-attachments/assets/3527fffc-8b49-41c5-b534-70b703e92beb" />

<img width="3823" height="1342" alt="step-11" src="https://github.com/user-attachments/assets/5ed80d2d-d71d-4c9e-b356-08ea09417c96" />

### Step 8: Add Payload Processing Rules to Forge Each Cookie

Add three payload processing rules, in this order, so every candidate password is transformed into a complete forged cookie:

1. **Hash → MD5** — hash the candidate password.
2. **Add prefix → `carlos:`** — prepend the victim's username and colon.
3. **Encode → Base64-encode** — produce the final cookie value.

Also change the request's `id` parameter from `wiener` to `carlos`, since the account page is requested for the victim.

<img width="2548" height="1272" alt="step-12" src="https://github.com/user-attachments/assets/68acc26d-f4cc-43cb-862f-5f355ef143a2" />

<img width="2235" height="1114" alt="step-13" src="https://github.com/user-attachments/assets/2e23a384-287c-4485-ac32-2727256f159a" />

<img width="2409" height="1212" alt="step-14" src="https://github.com/user-attachments/assets/f946ef9e-27d4-4609-bdee-d45e0eb7410c" />

<img width="3750" height="1576" alt="step-15" src="https://github.com/user-attachments/assets/aeb6d758-f285-47d6-9185-7d33e8835f38" />

### Step 9: Run the Attack and Find the 200 Response

Start the attack. Almost every forged cookie is rejected with a `302` response of length 173 (a redirect back to login). One request stands out: it returns **`200`** with a much larger response (length 3450) — that payload is Carlos's real cookie, built from his actual password.

<img width="3320" height="1720" alt="step-16" src="https://github.com/user-attachments/assets/8b5e5fc0-8361-4b90-93a5-1bbaa35d3370" />

### Step 10: Access Carlos's Account

Use the winning cookie value in the browser (or simply note that Intruder's successful request already reached the account page as Carlos). The lab is solved — access to the victim's **My account** page was gained without ever logging in with his password directly.

<img width="3228" height="1547" alt="step-17" src="https://github.com/user-attachments/assets/b1a34830-3c4a-406e-9f98-e1ceadc5e714" />

## Key Takeaways

- A persistent cookie derived from `username + MD5(password)` is not a session token — it is a reusable, forgeable credential. Anyone who learns the format can mint cookies offline.
- MD5 is fast and unsalted, so candidate passwords can be hashed and encoded at scale; Intruder's payload processing rules (hash → prefix → encode) automate the entire forgery chain.
- Removing the normal session cookie during testing proves the stay-logged-in cookie alone is sufficient for authentication — which is exactly why its weakness is critical.

## Remediation

- "Remember me" cookies must be long, random, unguessable tokens generated server-side, stored hashed in the database, and unrelated to the user's password.
- Never derive persistent authentication tokens from passwords or password hashes.
- Support token revocation and rotation, expire tokens after a reasonable period, and invalidate them on password change or logout.
