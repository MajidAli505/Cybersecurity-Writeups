# User ID Controlled by Request Parameter with Unpredictable User IDs

**Platform:** PortSwigger Web Security Academy
**Category:** Access Control
**Difficulty:** Apprentice
**Status:** ✅ Solved

## Background

Applications often replace simple sequential IDs (`1`, `2`, `3`) with GUIDs/UUIDs, such as `/user/8f14e45f-ea3b-4d9f-9d4c-123456789abc`, so that an attacker cannot guess another user's identifier. That only raises the cost of *finding* an ID — it changes nothing about *authorization*. If another user's GUID shows up anywhere in the application — messages, comments, reviews, profile links, APIs — an attacker can take it and use it exactly like a sequential ID. The line worth remembering: **GUIDs make guessing difficult, but they are not a replacement for authorization. If a GUID leaks and there is no access check, the vulnerability is still there.**

## Lab Objective

This lab has a horizontal privilege escalation vulnerability on the user account page, but identifies users with GUIDs. Find the GUID for `carlos`, then submit his API key as the solution.

**Provided resources:** your credentials `wiener:peter`.

<img width="2191" height="849" alt="step-1" src="https://github.com/user-attachments/assets/e3af3459-cbb7-4e3a-87a0-332f5fc5bba4" />


## Walkthrough

### Step 1: Log In and Note Your Own GUID

Log in using the supplied credentials and open your account page. As in the simpler variant of this lab, the page is selected by an `id` parameter — but this time the value is a GUID rather than your username:

```http
GET /my-account?id=<your GUID>
```

The page displays your username and API key. Guessing `carlos`'s GUID is hopeless, so the attack hinges on finding it disclosed somewhere instead.

<img width="3167" height="1251" alt="step-2" src="https://github.com/user-attachments/assets/c6502408-e4f7-461a-8d3d-4dff79c61976" />


<img width="2478" height="1299" alt="step-3" src="https://github.com/user-attachments/assets/5573be03-19ed-4a2b-8807-69a0e1b5e71a" />


<img width="2719" height="1556" alt="step-4" src="https://github.com/user-attachments/assets/de2b1993-53ca-44e0-93e2-0ff56d3de141" />


### Step 2: Find carlos's GUID in a Blog Post

Browse the site's blog and open a post written by `carlos`. In the page source, the author link is built from his user ID:

```html
<a href="/blogs?userId=<carlos's GUID>">carlos</a>
```

The unguessable identifier is printed in public markup for anyone who views the source. Copy it.

<img width="2861" height="1744" alt="step-5" src="https://github.com/user-attachments/assets/7d346ad1-e6d0-4432-859e-356f8052a247" />


<img width="2674" height="1190" alt="step-6" src="https://github.com/user-attachments/assets/98df6509-d5f2-4382-84a2-598580f903b4" />


### Step 3: Open carlos's Account Page

Log in as `wiener` and request your account page with `carlos`'s GUID in place of your own:

```http
GET /my-account?id=<carlos's GUID>
```

The server performs the same missing check as before: it serves the account belonging to the supplied GUID, including `carlos`'s API key, under your session.

<img width="2687" height="1638" alt="step-7" src="https://github.com/user-attachments/assets/511d21d9-b87c-46db-bc8a-0b0bdb99a471" />


<img width="3016" height="1537" alt="step-8" src="https://github.com/user-attachments/assets/81006308-8d15-4cc1-b3a5-3745b6556a37" />


### Step 4: Submit the API Key

Submit the API key from `carlos`'s account page using the lab's "Submit solution" button. The lab is solved.

<img width="3037" height="1535" alt="step-9" src="https://github.com/user-attachments/assets/2817f37e-21a4-4373-8aea-a05baf621dd7" />


## Key Takeaways

- Unpredictable identifiers are a speed bump, not a control: they stop enumeration, not use. Authorization must be checked independently of how hard an ID is to guess.
- GUIDs leak constantly through ordinary features — author links, comments, shared URLs, API responses. Any leak converts "unpredictable" back into "known".
- The vulnerability never changed between the two labs: the account page serves whichever user the parameter names, without binding it to the session. Only the ID format did.

## Remediation

Check authorization on every request for account data: the record being accessed must belong to the authenticated session (or the session must be an administrator). Treat GUIDs as a defense-in-depth measure against enumeration, never as the access control itself — and avoid exposing user GUIDs in public links and markup where they serve no purpose.
