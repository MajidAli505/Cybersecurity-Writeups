# Overheard at Breakfast

**Platform:** TryHackMe
**Category:** Challenge Room (OSINT)
**Difficulty:** Easy
**Status:** ✅ Solved (flag captured)

## Background

Overheard at Breakfast is an **OSINT** room with no target machine at all — the evidence is a single screenshot of a chat between two people, and everything needed is said out loud in it. One of them ("Lambo") drops two clues: enough to identify his email identity, and the fact that he links **all his social media accounts** through a free online profile tool **whose name starts with "G"** — **Gravatar**. Gravatar profiles are addressed by the **MD5 hash of the account's email address**, so the email from the chat converts directly into his public profile URL. The profile gathers his linked accounts and details in one place — and one item on it is **Base64-encoded**. Decode it, and the flag falls out. The room's lesson is the OSINT core skill: people narrate their own digital footprint; the investigator just connects the identifiers.

## Lab Objective

Extract the two clues from the chat screenshot, identify the profile service, build the target's Gravatar profile URL from the MD5 of his email, and decode the Base64 string on the profile to get the flag.

**Provided resources:** one task file — a PNG screenshot of a two-person chat.

<img width="2811" height="917" alt="step-1" src="https://github.com/user-attachments/assets/2f99ba03-4a0b-41f7-b9da-b8b32f4484e8" />

<img width="3348" height="1573" alt="step-2" src="https://github.com/user-attachments/assets/25db1945-5bad-4d9e-8cc2-e01d15c857e4" />

<img width="2049" height="1668" alt="step-3" src="https://github.com/user-attachments/assets/bbb3ecf2-87d0-4646-a5ca-2bb60648b574" />

## Walkthrough

### Step 1: Read the Chat Screenshot Closely

Download the task file: it is a **PNG of a chat conversation** between two people. Read it like an investigator, not a reader — two clues matter. One establishes **Lambo's email identity** (the address itself appears in the screenshots and is not repeated here); the other is about where his accounts are gathered.

<img width="2586" height="731" alt="step-4" src="https://github.com/user-attachments/assets/1d23e6fb-b987-4852-b604-7019fbdf131c" />

### Step 2: Identify the "G" Tool — Gravatar

In the chat, Lambo says he uses a **free online tool** where he links all his social media accounts into one profile, and that the tool's name **starts with G**. That description is **Gravatar** — the avatar/profile service that ties a person's accounts to their email address.

<img width="2870" height="924" alt="step-5" src="https://github.com/user-attachments/assets/706406dc-93f5-469e-bd67-e4349e0dc7e7" />

### Step 3: Build the Profile URL — MD5 of the Email

Gravatar does not put the email itself in a profile link; a profile is addressed by the **MD5 hash of the email address**. So generate exactly that from the email recovered in Step 1:

```bash
echo -n "the-email-from-the-chat" | md5sum
```

(`echo -n` matters: a trailing newline would change the hash.)

<img width="2217" height="262" alt="step-6" src="https://github.com/user-attachments/assets/99b5af10-452a-4209-b43c-4ab8cd773397" />

<img width="1803" height="616" alt="step-7" src="https://github.com/user-attachments/assets/af36b53f-d61d-4ff4-8f55-720dbf2f69bc" />

### Step 4: Open the Profile

Append the hash to the Gravatar address — `https://gravatar.com/<md5-hash>` — and open it. **Lambo's profile loads**, with his linked accounts and profile details collected in one place, exactly as the chat promised.

<img width="3297" height="1647" alt="step-8" src="https://github.com/user-attachments/assets/38584621-c309-4de6-9871-39ab9ff18e5d" />

### Step 5: Decode the Base64 — Flag

Among the profile's details, one string is plainly **Base64-encoded** — long, alphabet-and-`+`/`/` characters with the tell-tale shape. Decode it (CyberChef's "From Base64", or `base64 -d`), and the plaintext is the **flag** (value in the screenshot, not repeated here).

<img width="1814" height="1438" alt="step-9" src="https://github.com/user-attachments/assets/a8a95ac9-19be-4da9-98d6-51eb22a17b4e" />

## Key Takeaways

- OSINT starts with reading, not tooling: the entire chain — identity, service, profile — was stated in one casual conversation. Screenshots, chats and posts are evidence; extract every identifier they contain.
- Identifiers connect services. An email address became a Gravatar URL through a documented convention (MD5 of the address) — knowing how platforms address their objects is what turns a clue into a location.
- Aggregator profiles are exposure multipliers: one Gravatar page gathered every account the target had linked, doing the investigator's correlation work for him.
- Encodings are not encryption: a Base64 string on a public profile is an open envelope. Recognising common encodings (Base64's alphabet and padding) is a core triage skill.

## Remediation

The defensive reading is about footprint discipline: assume any public aggregator profile correlates your accounts for anyone who finds one identifier, and that email addresses — shared casually in chats and screenshots — are the keys that unlock those correlations. Keep personal and role identities separate, avoid linking every account into a single public profile, never post encoded "secrets" publicly (encoding hides nothing), and review what chat screenshots reveal before sharing them — names, addresses and habits are all intelligence to someone collecting them.
