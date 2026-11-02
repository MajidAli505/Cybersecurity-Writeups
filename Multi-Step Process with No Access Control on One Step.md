# Multi-Step Process with No Access Control on One Step

**Platform:** PortSwigger Web Security Academy
**Category:** Access Control
**Difficulty:** Practitioner
**Status:** ✅ Solved

## Overview

Sensitive operations are often split into steps: choose the action, then confirm it. Developers sometimes place the authorization check on the first step only, assuming nobody can reach step two without passing step one. That assumption is false on the web — every step is just a request, and any request can be sent directly. If the final, state-changing step performs no check of its own, it will happily execute for anyone who asks, and the careful check at the entrance guards nothing.

## Lab Objective

This lab has an admin panel with a flawed multi-step process for changing a user's role. Log in as `wiener:peter` and exploit the flawed access controls to promote yourself to become an administrator.

**Provided resources:** you can familiarize yourself with the admin panel first using the credentials `administrator:admin`.

<img width="2176" height="1054" alt="step-1" src="https://github.com/user-attachments/assets/e5d31cc9-fd13-4e1f-abe7-6b0aa08ada62" />

## Walkthrough

### Step 1: Walk the Genuine Process and Capture the Confirmation Request

Log in using the admin credentials and start upgrading a user in the admin panel. The process has two steps: step 1 submits the chosen user and action (`POST /admin-roles` with `username` and `action=upgrade`), and step 2 is the confirmation — an "Are you sure?" page whose submission actually applies the change. Capture the step-2 request, the one that confirms the upgrade.

<img width="3183" height="1825" alt="step-2" src="https://github.com/user-attachments/assets/5cf77988-5cb9-4d10-84e5-06bc692dc625" />
<img width="2078" height="539" alt="step-3" src="https://github.com/user-attachments/assets/866f081c-9799-43ae-beb0-0d78dca82f38" />
<img width="2414" height="942" alt="step-4" src="https://github.com/user-attachments/assets/a8d110ff-9828-4633-a2a9-9549d9df5f17b" />

<img width="2677" height="1515" alt="step-5" src="https://github.com/user-attachments/assets/51d4a5ae-35dd-41d0-b281-be262cee47fe" />

### Step 2: Get a Non-Admin Session

Now log in with the non-administrator account (`wiener:peter`) and, using the browser's developer tools or Burp, copy that user's session cookie. This ordinary session is all the attack needs.

<img width="3094" height="1342" alt="step-6" src="https://github.com/user-attachments/assets/fc01e3cc-3e72-4bd4-a45b-3090b5617d09" />
<img width="3445" height="1738" alt="step-7" src="https://github.com/user-attachments/assets/7baf221d-4241-4fea-a4cd-657ccabfd1e5" />

### Step 3: Replay the Confirmation with wiener's Session

Take the captured step-2 request and modify it in two places: replace its session cookie with the non-administrator's session cookie, and set the username in the request to `wiener`. Send it, and in Burp click "Follow redirection" to complete the flow:

```http
POST /admin-roles HTTP/2
Host: <lab host>
Cookie: session=<wiener's session>

username=wiener&action=upgrade&confirmed=true
```

The confirmation step never asks whether the session belongs to an administrator — it simply applies the role change it was handed.

<img width="2787" height="1298" alt="step-8" src="https://github.com/user-attachments/assets/7bf41dbf-b3ba-46d9-9a3f-e09408161241" />
<img width="2691" height="1435" alt="step-9" src="https://github.com/user-attachments/assets/40ad87ba-5933-4b28-9e14-03d452da73bf" />
<img width="2830" height="1455" alt="step-10" src="https://github.com/user-attachments/assets/56c510b0-40a4-4799-9bb2-9a239160ff54" />

### Step 4: Confirm the Promotion

The flow lands on the confirmation page and the upgrade is applied: `wiener` is now an administrator, the admin panel is available to his account, and the lab is solved.

<img width="3159" height="1123" alt="step-11" src="https://github.com/user-attachments/assets/3adcef41-7f75-4713-989a-401303d26d81" />

## Key Takeaways

- Every step of a multi-step process is independently reachable. Authorization checked at step one and assumed at step two is authorization that does not exist.
- The dangerous request is the one that changes state — that is precisely the request that must verify the caller's privileges, every time.
- A supplied admin account for "familiarization" doubles as a request-recording tool: capture the genuine privileged request, then replay it under a low-privilege session to see which steps actually enforce anything.

## Remediation

Authorize every request in a workflow, especially the final state-changing one; never infer authorization from the fact that a user "must have" passed an earlier step. Server-side, bind the steps of a transaction together (for example, with a server-issued, session-scoped token created only after the initial check) and re-validate the session's privileges when the change is applied.
