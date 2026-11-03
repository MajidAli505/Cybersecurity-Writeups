# Authentication Bypass via Flawed State Machine

**Platform:** PortSwigger Web Security Academy
**Category:** Business Logic
**Difficulty:** Practitioner
**Status:** ✅ Solved

## Background

Some applications model authentication as a **state machine**: the session moves through states — *anonymous → credentials verified → role selected → fully authenticated* — and each request is supposed to advance it one step. The flaw appears when the application assumes the machine can only be in a sensible state, and fails to handle a session that stops halfway. If a user authenticates but never completes the role-selection step, the session has **no role at all** — and in this application, downstream code treats "no role assigned yet" with a default that turns out to be the most privileged one. Skipping a step becomes an escalation.

## Lab Objective

Exploit the flawed authentication state machine to gain administrative access, then delete the user `carlos`.

**Provided resources:** your credentials `wiener:peter`.

<img width="3043" height="1761" alt="step-1" src="https://github.com/user-attachments/assets/203b657f-0259-45ee-9ec8-b8a9aa6a0104" />

## Walkthrough

### Step 1: Map the Login Workflow

Log in through Burp and watch the sequence. Authentication here is two-staged: `POST /login` verifies the credentials and responds with a redirect to a **role-selector** page (`Location: /role-selector`), where the user picks which role to act as (for example, *content author*). Only after that second step is the session fully established with a role.

<img width="2983" height="1349" alt="step-2" src="https://github.com/user-attachments/assets/d11f8603-9f1b-43c9-9757-8ddf389718f7" />

<img width="3077" height="912" alt="step-3" src="https://github.com/user-attachments/assets/1824353c-40e7-4824-8015-54737e7c40b3" />

<img width="2991" height="1356" alt="step-4" src="https://github.com/user-attachments/assets/9e3a517d-bb88-40ba-9e80-37c94e61e530" />

### Step 2: Try Modifying the Role-Selection Request

The direct attack is to tamper with the role-selector request itself — change the chosen role to something more powerful. The application rejects this with an error: the roles offered to this account are enforced on that request. The front door to privilege is guarded.

<img width="2994" height="1453" alt="step-5" src="https://github.com/user-attachments/assets/81e73a1e-6197-4fac-af0f-78e395bddc7d" />

<img width="2881" height="1265" alt="step-6" src="https://github.com/user-attachments/assets/b122738e-ea58-499a-9548-f669be178064" />

### Step 3: Abandon the State Machine Mid-Flight

Log out and log in again, this time controlling the flow in Burp's proxy:

1. Forward the `POST /login` request. Credentials are verified, a fresh authenticated session cookie is issued, and the server redirects toward `/role-selector`.
2. **Drop the role-selector request** — simply never let the role-selection step happen.
3. Now browse the site directly with that session.

The session exists and is authenticated, but no role was ever written into it. When the application evaluates what this session may do, the uninitialized role falls through to a default — and that default is **administrator**. You are in with full privileges, without any role ever having been granted to you.

<img width="2643" height="792" alt="step-7" src="https://github.com/user-attachments/assets/46a939af-e5ce-41a4-8998-489daf00b426" />

### Step 4: Delete carlos

With the accidental administrator session, open the admin panel and delete `carlos`. The lab is solved.

## Key Takeaways

- Uninitialized state is a state too — and attackers specialize in reaching it. Any value your code reads must have a safe value in *every* reachable state, including half-finished workflows.
- Defaults must fail closed. If a missing role had defaulted to "no access", the skipped step would have produced a dead session instead of an admin.
- When testing multi-stage flows, don't only tamper with values — tamper with the *sequence*: drop requests, abandon flows midway, and reuse sessions from incomplete states.

## Remediation

Initialize sessions to the least-privileged state at creation, and require every protected resource to check for an explicitly granted role rather than the absence of a denial. Treat incomplete workflows as unauthenticated: if a required stage hasn't completed server-side, the session should gain nothing. Where a state machine is used, enumerate its transitions and make every state — including transitional ones — safe by construction.
