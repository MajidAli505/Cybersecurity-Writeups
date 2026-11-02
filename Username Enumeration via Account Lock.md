# Username Enumeration via Account Lock

**Platform:** PortSwigger Web Security Academy  
**Category:** Authentication Vulnerabilities  
**Difficulty:** Apprentice  
**Status:** ✅ Solved

## Overview

Websites often try to prevent brute-force attacks by locking an account after a set number of failed login attempts. However, just as ordinary login error messages can leak information, a response indicating that an account has been locked can also help an attacker enumerate valid usernames — because the application only tracks failed attempts, and therefore only locks, accounts that actually exist.

## Lab Objective

Enumerate a valid username by triggering the account lock, then brute-force that user's password and log in.

**Provided resources:** a candidate list of usernames and a candidate list of passwords.

## Walkthrough

### Step 1: Capture the Login Request

With Burp Suite running, investigate the login page and submit an invalid username and password. Send the `POST /login` request to Burp Intruder.

### Step 2: Set Up a Cluster Bomb Attack to Repeat Every Username

Select the **Cluster bomb** attack type. Add a payload position to the `username` parameter, and add a blank payload position at the end of the request body by clicking **Add §**. The result should look like this:

```text
username=§invalid-username§&password=<redacted>
```

### Step 3: Configure the Payloads

In the **Payloads** side panel, load the list of usernames into the first payload position. For the second payload position, select the **Null payloads** type and choose the option to generate **5 payloads**. This effectively repeats each username 5 times. Start the attack.

### Step 4: Identify the Valid Username

In the results, notice that the responses for one username are longer than the responses for the others. Studying that response more closely reveals a different error message:

```text
You have made too many incorrect login attempts.
```

This is the account-lock message — it only appears for a real account. Make a note of this username.

### Step 5: Brute-Force the Password

Create a new Burp Intruder attack on the `POST /login` request, this time selecting the **Sniper** attack type. Set the `username` parameter to the username identified in Step 4 and add a payload position to the `password` parameter.

### Step 6: Run the Password Attack with Grep Extract

Load the list of passwords into the payload set, and create a grep extraction rule for the error message. Start the attack.

### Step 7: Identify the Correct Password

In the results, look at the grep extract column. There are a couple of different error messages, but one response contains **no error message at all** — that is the successful login attempt. Make a note of this password.

### Step 8: Log In

Wait for a minute to allow the account lock to reset. Then log in using the username and password you identified, and access the user account page to solve the lab.

## Why This Works

**Null payloads as a counter:** A null payload makes Burp inject nothing at that position — it is used purely to repeat the request. With null payloads set to 5, Burp sends 5 requests for every username, always with the same password (`example`):

```text
adam / example    (x5)
carlos / example  (x5)
wiener / example  (x5)
```

**Why exactly 5 attempts:** The lab's lockout policy is:

```text
5 failed attempts → account locked
```

If the username exists, the responses change at the threshold:

```text
1st–4th request → Invalid username or password
5th request     → Too many incorrect login attempts
```

If the username does not exist, every response stays `Invalid username or password` and the account is never locked, because the system only counts failed attempts against real accounts. That difference is the enumeration oracle.

**Why Cluster Bomb:** Cluster Bomb combines the two payload sets — the usernames list and the 5 null values — so Burp automatically tests each username exactly 5 times. The null payload set acts as a counter rather than as injected data.

## Screenshots

<img width="3111" height="1515" alt="step-1" src="https://github.com/user-attachments/assets/d2110073-9529-488b-be76-0b2fe1a7d974" />

<img width="3109" height="1291" alt="step-2" src="https://github.com/user-attachments/assets/df85963e-6b6d-4a8a-9d3b-f0ef0fe344f5" />

## Key Takeaways

- Anti-brute-force controls can themselves become an information leak: any response that differs between existing and non-existing accounts enables username enumeration.
- Repeating each candidate username exactly up to the lockout threshold (using null payloads as a counter) turns the lockout message into a reliable validity oracle.
- Once a valid username is known, a second, focused attack (Sniper) with response-based grep extraction efficiently identifies the correct password.

## Remediation

- Return an identical, generic error (e.g. "Invalid username or password") in all failure cases, including when an account is locked.
- Apply rate limiting and lockout in ways that do not reveal whether an account exists — for example, throttling by IP or device in addition to per-account controls, and informing the legitimate user of a lockout through a separate channel such as email.
