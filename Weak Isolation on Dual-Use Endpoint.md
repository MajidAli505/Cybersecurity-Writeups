# Weak Isolation on Dual-Use Endpoint

**Platform:** PortSwigger Web Security Academy
**Category:** Business Logic
**Difficulty:** Practitioner
**Status:** ✅ Solved

## Background

A dual-use endpoint serves two different audiences through one URL — here, the change-password endpoint is used both by ordinary users changing their own password and by a privileged flow where no current password is required. The endpoint decides which rules apply by looking at the shape of the request: if a `current-password` parameter is present, it runs the normal-user logic and checks it; if the parameter is absent, it assumes the request belongs to the privileged flow and skips the check. That is a flawed assumption about the caller's privilege level based purely on their input — because any ordinary user can simply leave the parameter out and be treated as the privileged kind of caller.

## Lab Objective

This lab makes a flawed assumption about the user's privilege level based on their input. Exploit the logic of its account management features to gain access to arbitrary users' accounts — specifically, access the `administrator` account and delete the user `carlos`.

**Provided resources:** your credentials `wiener:peter`.

<img width="2215" height="863" alt="step-1" src="https://github.com/user-attachments/assets/5d1c09c4-2e1e-4e74-8a94-bd46751a1458" />

## Walkthrough

### Step 1: Try the Direct Approach

Log in and open the change-password feature. The obvious attack — set the username field to `administrator` and choose a new password — fails with an error: as a normal user, you are required to prove knowledge of the account's *current* password, which you don't have.

<img width="3105" height="1522" alt="step-2" src="https://github.com/user-attachments/assets/7f8d62b8-5f6b-4be1-90f6-c836d0892236" />

<img width="3351" height="1907" alt="step-3" src="https://github.com/user-attachments/assets/9d555820-4bec-43d9-bc3c-5401238313ff" />

<img width="3081" height="1629" alt="step-4" src="https://github.com/user-attachments/assets/06ed1824-6e3b-42b8-b27e-1c78115749ef" />

### Step 2: Study the Change-Password Request

Capture the request and send it to Burp Repeater. Its body carries the expected fields:

```http
POST /my-account/change-password HTTP/2
Host: <lab host>
Cookie: session=<your session>

username=administrator&current-password=<value>&new-password-1=<new>&new-password-2=<new>
```

The check that defeated Step 1 hangs entirely on that one parameter: `current-password`.

<img width="2799" height="1776" alt="step-5" src="https://github.com/user-attachments/assets/9f96110b-53c9-40c2-ae74-ad68346d1fe2" />

### Step 3: Remove the Parameter Entirely

Delete the `current-password` parameter from the request altogether — don't empty it, remove it — and send the request again:

```http
username=administrator&new-password-1=<new>&new-password-2=<new>
```

This time the response is a 302 success. With the parameter missing, the endpoint switched to its other personality: the privileged, dual-use variant that resets a password without asking for the old one. It never verified that the session making this request was entitled to use that variant.

<img width="2441" height="1508" alt="step-6" src="https://github.com/user-attachments/assets/a0ae9fab-9b61-4539-9727-60d8908fffc9d" />

<img width="2905" height="1441" alt="step-7" src="https://github.com/user-attachments/assets/2924eeeb-e041-4b38-bf7f-62f98879631c" />

### Step 4: Log In as Administrator and Delete carlos

Log in as `administrator` with the new password you just set. The admin panel is available. Delete `carlos` — the lab is solved.

<img width="3048" height="1235" alt="step-8" src="https://github.com/user-attachments/assets/3a8dc5ed-ffa4-4b69-b808-5428cbd78201" />

<img width="2988" height="882" alt="step-9" src="https://github.com/user-attachments/assets/6bc088df-f202-4509-87ac-c0ffb5ca0898" />

<img width="3099" height="1156" alt="step-10" src="https://github.com/user-attachments/assets/2fbf5523-033a-4e0a-a5f0-ab95f4e09550" />

## Key Takeaways

- Behavior selected by the *presence or absence* of a parameter is behavior the attacker selects. Optional parameters must never gate security checks.
- The endpoint's two audiences had different trust levels but shared one code path with weak isolation between them — the low-privilege audience inherited the high-privilege one's shortcuts.
- A missing-parameter case is exceptional input too: when testing, remove each parameter in turn and watch whether the application's assumptions change.

## Remediation

Isolate privileged functionality on its own endpoint, protected by a server-side authorization check based on the session's actual role — never on the shape of the request. For shared functionality, determine the caller's privilege from the authenticated session, and apply the strictest relevant checks (such as requiring the current password) whenever the session doesn't independently prove the higher privilege.
