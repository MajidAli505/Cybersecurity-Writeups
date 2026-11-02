# Insecure Direct Object References

**Platform:** PortSwigger Web Security Academy
**Category:** Access Control
**Difficulty:** Apprentice
**Status:** ✅ Solved

## Background

Insecure direct object references (IDORs) are a subcategory of access control vulnerabilities. An IDOR arises when an application uses user-supplied input to access an object directly — a file, a record, a document — and never checks whether the requester is entitled to that object. Because the reference is just an identifier the user can edit, modifying it yields unauthorized access. It is one of the most common implementation mistakes behind broken access control.

## Lab Objective

This lab stores user chat logs directly on the server's file system, and retrieves them using static URLs. Find the password for the user `carlos`, and log into his account.

<img width="2155" height="784" alt="step-1" src="https://github.com/user-attachments/assets/4325307d-4a1d-4425-be6c-d1d3d42f0124" />

## Walkthrough

### Step 1: Create a Chat Transcript and Study Its URL

Open the live chat, exchange a couple of messages with the support bot, and then use the option to view/download the transcript. The transcript is saved as a file and your browser downloads it. Watch the request in Burp: the transcript lives at a static URL whose only variable part is a sequential file name, along the lines of:

```http
GET /download-transcript/2.txt
```

A predictable, sequential reference to a stored object — and no session check standing behind it.

<img width="2960" height="1704" alt="step-2" src="https://github.com/user-attachments/assets/3aa55ef0-4090-4fb0-9765-d9a05d9480bb" />

<img width="3150" height="1552" alt="step-3" src="https://github.com/user-attachments/assets/1fbf8a7e-c646-418d-8eba-86735f3d3d246" />

<img width="2704" height="1754" alt="step-4" src="https://github.com/user-attachments/assets/565f10f0-2e10-4ad7-bdb7-789e1bfc4e9f" />

### Step 2: Walk the Sequence

Edit the file name in the URL and request other transcripts — for example, changing the ending to `/7.txt` confirms that arbitrary numbers return other users' files. Work down to the earliest one:

```http
GET /download-transcript/1.txt
```

<img width="2515" height="1318" alt="step-5" src="https://github.com/user-attachments/assets/ba3520bb-d74e-44f4-8366-4ca77ac04604" />

### Step 3: Read carlos's Password

Open `1.txt`. It is a transcript of `carlos`'s own support conversation, in which he gave away his account password. The file was never meant for you — nothing checked whose transcript you were asking for; the server simply served the file named in the URL. (The password itself is visible in the screenshot below.)

<img width="2988" height="1184" alt="step-6" src="https://github.com/user-attachments/assets/51c219a8-95f1-4bcf-924d-631dc5181b37" />

### Step 4: Log In as carlos

Log in with the username `carlos` and the recovered password. His account page loads — you are in as `carlos`, and the lab is solved.

<img width="2315" height="1439" alt="step-7" src="https://github.com/user-attachments/assets/9e1fb8cd-acd3-4331-a3c1-8c257a9d4ff1" />

<img width="3127" height="1455" alt="step-8" src="https://github.com/user-attachments/assets/dd5c1d62-878f-488e-834f-262c990d2201" />

## Key Takeaways

- A direct reference plus a missing ownership check is the whole vulnerability: the server answered "which file?" without ever asking "whose file?".
- Sequential identifiers make exploitation trivial — every neighboring object is one increment away. Unpredictable names would slow this down, but only a real authorization check stops it.
- Stored content is often richer than expected: a chat transcript handed over a live credential, turning a privacy leak into account takeover.

## Remediation

Serve private files through an authenticated handler that verifies the requester owns (or may access) the specific object before returning it — never expose user content at static, guessable URLs. Indirect references (mapping a per-session token to the object) can reduce exposure, but they supplement authorization checks rather than replacing them.
