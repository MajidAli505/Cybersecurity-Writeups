# Broken Brute-Force Protection, Multiple Credentials per Request

**Platform:** PortSwigger Web Security Academy
**Category:** Authentication Vulnerabilities
**Difficulty:** Expert
**Status:** ✅ Solved

## Overview

This lab has broken brute-force protection: the login form accepts **multiple credentials in a single request**.

The application applies rate limiting to login attempts, and the victim's username is given in the lab: **`carlos`**. The objective is to find Carlos's password and log in to his account. The application counts *requests*, not *password guesses* — and because it accepts credentials in JSON format, an entire password list can be submitted as a JSON array in one single request, so the rate limiter sees one attempt while the server checks every password in the list.

## Key Concepts

- **Account lockout** locks an *account* after a set number of failed attempts. It can be sidestepped by password spraying (a few common passwords against many accounts) and by credential stuffing (one stolen credential pair per account — a single attempt never triggers a lockout), and it can even be abused to lock legitimate users out of their accounts.
- **Rate limiting** blocks an *IP address* after a set number of requests. It is generally safer than account lockout because the victim's account stays usable, but it fails if the attacker can change or spoof their apparent IP address — for example through IP rotation or an `X-Forwarded-For` header the application trusts.
- **The flaw in this lab is different:** the rate limiter counts HTTP requests, while the application itself will process *many* password guesses inside one request if they are supplied as a JSON array. One request can therefore equal one hundred guesses.

## Walkthrough

### Step 1: Confirm the Rate Limit — and That IP Spoofing Does Not Help

The lab told me the victim's username is `carlos`; I needed to find his password. I first tried IP spoofing to bypass the rate limit, but it didn't work:

<img width="2993" height="1296" alt="step-1" src="https://github.com/user-attachments/assets/c68eabed-5bf4-41b7-a261-36f6db38c5f" />

### Step 2: Notice the Credentials Are Sent as JSON

Looking at the login request, I saw that the credentials were sent in JSON format. That suggested the server might accept JSON arrays:

<img width="2070" height="1022" alt="step-2" src="https://github.com/user-attachments/assets/2a8e0ea1-0c2d-4797-af2f-58fcfb1e2c5c" />

### Step 3: Convert the Password Wordlist to a JSON Array

I had a wordlist of 100 candidate passwords for the lab. First, I converted the wordlist into JSON format:

<img width="2012" height="1330" alt="step-3" src="https://github.com/user-attachments/assets/d5c7c397-f6c8-4ef2-8f7e-74350d7d4ad4" />

<img width="1529" height="1112" alt="step-4" src="https://github.com/user-attachments/assets/bcce9046-a58f-4c67-af20-d4f91a2d882e" />

### Step 4: Send All 100 Passwords in a Single Request

I pasted the passwords into the `password` field of the login request, in JSON array format, and sent a single request:

<img width="3004" height="1551" alt="step-5" src="https://github.com/user-attachments/assets/4a073718-a829-4c6c-b124-4fa86ba8bd73" />

### Step 5: Open the Successful Response in the Browser

In Burp, I right-clicked the successful request and selected **Open response in browser**:

<img width="3180" height="1382" alt="step-6" src="https://github.com/user-attachments/assets/cdc78f00-1f70-4252-a103-f30ce839c07d" />

I then copied the link from the popup:

<img width="2299" height="999" alt="step-7" src="https://github.com/user-attachments/assets/ba5f26b4-90f5-4c15-b22b-399db4379c97" />

### Step 6: Log In as Carlos

I pasted the link into the browser. The page loaded and I was logged in as **`carlos`**, which solved the lab:

<img width="3194" height="1503" alt="step-8" src="https://github.com/user-attachments/assets/320646d1-d756-42e2-bddb-3976d086df2f" />

## Key Takeaways

- A rate limiter that counts *requests* provides no protection if a single request can carry many guesses.
- Accepting credentials in a flexible format (such as JSON) can silently change the security properties of an endpoint.
- Always validate and constrain the type and structure of user-supplied data, especially in security-critical endpoints such as login.
- Brute-force protection must consider the number of credentials processed, not just the number of requests received.

## Remediation

- Enforce a strict schema for login requests: reject requests where the `password` field is not a single string.
- Count each credential guess separately for rate limiting, regardless of how credentials are packaged.
- Combine request rate limiting with per-account protections (such as progressive delays or CAPTCHAs) rather than relying on IP-based limits alone.
