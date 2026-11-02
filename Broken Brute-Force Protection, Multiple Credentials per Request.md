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

The lab told me the victim's username is `carlos`; I needed to find his password. I first tried IP spoofing to bypass the rate limit, but that technique did not work here — the application applied the rate limit anyway, as shown on the right side of the screenshot:

[[IMAGE_1]]

### Step 2: Notice the Credentials Are Sent as JSON

Looking at the login request, I saw that the credentials were sent in JSON format. That suggested the second technique: **multiple credentials per request**.

[[IMAGE_2]]

### Step 3: Convert the Password Wordlist to a JSON Array

I had a wordlist of 100 candidate passwords for the lab. First, I converted the wordlist into JSON format:

[[IMAGE_3]]

[[IMAGE_4]]

### Step 4: Send All 100 Passwords in a Single Request

I pasted the passwords into the `password` field of the login request, in JSON array format, and sent the request. The response was a **`302` redirect** — the technique had worked, and one of the passwords in the array was correct. Because the whole list went in a single request, the rate limiter never blocked the attempt:

[[IMAGE_5]]

### Step 5: Open the Successful Response in the Browser

In Burp, I right-clicked the successful request and selected **Open response in browser**:

[[IMAGE_6]]

I then copied the link from the popup:

[[IMAGE_7]]

### Step 6: Log In as Carlos

I pasted the link into the browser. The page loaded and I was logged in as **`carlos`**, which solved the lab:

[[IMAGE_8]]

## Key Takeaways

- A rate limiter that counts *requests* provides no protection if a single request can carry many guesses. Defences must count **authentication attempts**, not HTTP requests.
- Accepting credentials in a flexible format (such as JSON) can silently change the security properties of an endpoint: type confusion — here, an array where a string is expected — turned the login form into a bulk password checker.
- Defence in depth matters: account lockout can be sprayed or stuffed around, and IP rate limiting can be rotated or spoofed around. Layered controls, combined with strict input validation, are what make brute-force protection hold.

## Remediation

- Validate credential fields strictly: the `password` field must be a single string, and any other type (such as an array) must be rejected.
- Apply rate limiting and lockout logic per *credential attempt*, including every value inside a multi-value submission.
- Combine IP-based rate limiting (based on the real connection IP, not client-supplied headers) with per-account throttling and alerting on credential-stuffing patterns.
