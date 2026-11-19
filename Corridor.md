# Corridor

**Platform:** TryHackMe
**Category:** Challenge Room
**Difficulty:** Easy
**Status:** ✅ Solved (flag captured)

## Background

Corridor is the smallest possible lesson in **IDOR with hashed identifiers**. The site is a corridor of doors: every room's page is addressed not by a plain number but by an **MD5 hash** in the URL — which looks like protection, and is none. Hashing a tiny, predictable value space is just encoding: CrackStation identifies the room hashes as the MD5 digests of the small integers `1, 2, 3, 4...`. Once the pattern is known, the interesting question is which number is *missing* from the corridor — `0`. Generate `MD5("0")` in CyberChef, put it in the URL, and the hidden room with the flag opens. The room's whole point: an identifier that anyone can compute is not an access control, no matter how long and random it looks.

## Lab Objective

Recognise that the corridor's URL hashes are MD5 digests of sequential small numbers, find the missing number, forge its hash, and open the hidden room for the flag.

**Provided resources:** the target machine only; no credentials are given.

<img width="2859" height="902" alt="step-1" src="https://github.com/user-attachments/assets/dade4e48-71fe-4b49-a389-92e4f3cba320" />


<img width="3151" height="774" alt="step-2" src="https://github.com/user-attachments/assets/a02e2989-bfe9-4a6f-82c3-c5c51a0e7b11" />


## Walkthrough

### Step 1: Browse to the Corridor

Open the target IP in the browser. The site is literally a **corridor** — a page of door/room images, each one a link to that room's own page.

<img width="2841" height="1683" alt="step-3" src="https://github.com/user-attachments/assets/703773ef-d824-4455-9d51-d2ab6c5972fc" />


### Step 2: Inspect the Page — the URLs Are Hashes

Inspect the page (view source / developer tools) and look at the links. Every room URL ends in a long hexadecimal string — a **hash** — instead of a readable name or number. Copy the links out; the pattern is the puzzle.

<img width="2088" height="1635" alt="step-4" src="https://github.com/user-attachments/assets/53a2d812-a22a-429b-b819-0fe480ba7245" />


### Step 3: Follow a Link

Use one of the links: it opens the picture inside that room. The hash in the address bar is now the working identifier — whatever the hash stands for, the server serves the room it names without any check.

<img width="2272" height="1188" alt="step-5" src="https://github.com/user-attachments/assets/47d77422-f1a6-4c83-8376-ffc46ff5539b" />


### Step 4: Crack the Hash — It Is Just the Number 1

Copy one hash and submit it to **CrackStation**. It answers immediately: the string is an **MD5** hash, and its plaintext is **`1`**. The "random" identifier is the MD5 of a small integer.

<img width="1826" height="1075" alt="step-6" src="https://github.com/user-attachments/assets/c375a04d-a75f-47b3-9cfe-65515c721496" />


<img width="2637" height="819" alt="step-7" src="https://github.com/user-attachments/assets/90415f81-eb7a-4789-b7ed-5391788d941e" />


### Step 5: Find the Gap in the Sequence

Check the remaining rooms' hashes the same way: they decode to **2, 3, 4...** — the corridor is numbered sequentially and every number's hash is computable by anyone. But one number never appears among the links: **0**. A hidden room addressed by `MD5("0")` is the obvious place for the flag — the developer hashed the IDs for obscurity and simply never linked room zero.

### Step 6: Forge the Missing Hash and Take the Flag

Generate the hash in **CyberChef** with the MD5 operation applied to the input `0`:

<img width="2889" height="1333" alt="step-8" src="https://github.com/user-attachments/assets/b123dc51-89e2-4b0e-8eb5-bfa58a97a2c8" />


Paste the resulting digest into the room URL in place of an existing hash and load it. The unlisted room opens — and the **flag** is inside (value in the screenshot, not repeated here).

<img width="2633" height="959" alt="step-9" src="https://github.com/user-attachments/assets/4856d64c-d7c7-4335-bcaa-99a66785b956" />


## Key Takeaways

- A hash of a predictable value is not a secret. MD5 of small integers is a lookup table the whole internet has already built — CrackStation answered from its index, no cracking required.
- IDOR is about authorisation, not identifier shape: sequential numbers, UUIDs and hashes all fail the same way when the server never checks whether the requester may see the object.
- Enumeration includes the values *between* the ones you are shown: rooms 1–4 were linked, and the flag sat at 0. Always probe the edges of a sequence — 0, 1, negative, and one past the end.
- CyberChef's hash generator plus an online hash database is the complete toolkit for this pattern: identify one sample, learn the scheme, mint the identifier you were never given.

## Remediation

Never use hashes of predictable values as identifiers for access control — obscurity is not authorisation. Serve objects only after a server-side check that the current user is entitled to them, whatever the URL looks like. Where unguessable references are genuinely needed, use random 128-bit identifiers (UUIDv4) *in addition to* authorisation checks, not instead of them, and avoid exposing sequential structure (counts, neighbours, first/last IDs) that tells an attacker exactly which values to try next.
