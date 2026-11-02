# Username Enumeration via Response Timing

**Platform:** PortSwigger Web Security Academy  
**Category:** Authentication Vulnerabilities  
**Status:** ✅ Solved

## Overview

This lab is vulnerable to username enumeration through **response timing**. The application also applies IP-based brute-force protection, which can be bypassed because it trusts the `X-Forwarded-For` header.

The idea behind the timing attack is subtle: when an **invalid** username is submitted, the server responds in roughly the same time every time. When a **valid** username is submitted with a very long password, the response takes noticeably longer — because the application only performs the expensive password-checking work for accounts that actually exist, and that work takes longer for a longer password. That timing difference is enough to identify a valid username, after which the password can be brute-forced in the normal way.

## Objective

To solve the lab, enumerate a valid username via response timing, brute-force that user's password, and log in to their account.

## Walkthrough

### Step 1: Test the Login Function and Find the IP Block

With Burp running, I submitted an invalid username and password and sent the resulting `POST /login` request to Burp Repeater. I experimented with different usernames and passwords, and observed that after too many invalid login attempts, my IP address was blocked.

### Step 2: Bypass the IP Block with `X-Forwarded-For`

I identified that the application supports the `X-Forwarded-For` header. By changing the value of this header, I could spoof my apparent IP address on each request and bypass the IP-based brute-force protection entirely.

### Step 3: Identify the Timing Difference

I continued experimenting with usernames and passwords, paying close attention to the response times:

- With an **invalid** username, the response time stayed roughly the same.
- With a **valid** username (I confirmed this using my own account), the response time **increased depending on the length of the password** submitted.

This confirmed that submitting a very long password would make a valid username stand out clearly in the timings.

### Step 4: Set Up the Username Enumeration Attack

I sent the request to Burp Intruder and configured a **Pitchfork** attack:

- I added the `X-Forwarded-For` header to the request.
- I added payload positions for the `X-Forwarded-For` header **and** the `username` parameter.
- I set the password to a very long string of characters (about 100 characters).

For the payloads:

- **Position 1** (`X-Forwarded-For`): the **Numbers** payload type, range 1–100, step 1, with max fraction digits set to 0. This spoofed a different IP address for every request.
- **Position 2** (`username`): the candidate username list provided for the lab.

Then I started the attack.

### Step 5: Identify the Valid Username

When the attack finished, I clicked **Columns** at the top of the results dialog and enabled the **Response received** and **Response completed** columns, which display the response timings in the results table.

One username's response time was significantly longer than all the others. I repeated that request a few times to confirm it consistently took longer, and noted the username.

### Step 6: Brute-Force the Password

I created a second Burp Intruder attack on the same request:

- I added the `X-Forwarded-For` header again, with a payload position on it (the numbers payload, as before).
- I fixed the `username` parameter to the username identified in Step 5.
- I added a payload position to the `password` parameter and loaded the candidate password list.

Then I started the attack.

### Step 7: Log In

When the attack finished, I looked for the response with a **`302` status code** — the successful login redirect — and noted the password that produced it.

Finally, I logged in with the identified username and password and accessed the user's account page, which solved the lab.

## Key Takeaways

- Username enumeration does not require different error messages. A consistent **difference in response time** leaks the same information: password verification work is only performed for accounts that exist, so a long password makes valid usernames measurably slower.
- Rate limiting and IP blocking based on the `X-Forwarded-For` header are not protections at all when the application trusts a header the client controls — the attacker simply presents a new "IP address" with every request.

## Remediation

- Perform the same password-verification work for every login attempt, whether or not the account exists (for example, by checking the supplied password against a dummy hash for unknown usernames), so response times do not reveal account validity.
- Return identical responses — and take comparable time — for all failed logins.
- Never base rate limiting or IP blocking on the `X-Forwarded-For` header unless it is set by a trusted proxy under your control; otherwise use the real connection IP address.
