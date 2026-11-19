# Neighbour

**Platform:** TryHackMe
**Category:** Challenge Room
**Difficulty:** Easy
**Status:** ✅ Solved (flag captured)

## Background

Neighbour is broken access control at its most literal. The room provides a login page, and its own source code tells you how to get in: an HTML comment instructs visitors to log in with the username and password **`guest`**. Once inside as the guest, the application identifies *whose* page you are viewing with a **`user` parameter in the URL** — and trusts it completely. Change `guest` to `admin` in the address bar and the server serves the admin's page, flag included, without ever asking for the admin's password. Two beginner lessons in one tiny room: view-source is reconnaissance, and any identity the *client* gets to state is an identity the client gets to choose.

## Lab Objective

Log in with the guest credentials disclosed in the page source, then change the user parameter in the URL to `admin` and capture the flag from the admin's page.

**Provided resources:** the target link; the guest credentials are discoverable in the page source.

<img width="2906" height="920" alt="step-1" src="https://github.com/user-attachments/assets/4c37cb74-85ff-4fb9-be51-8a6c0a0a5bcd" />


<img width="3273" height="1398" alt="step-2" src="https://github.com/user-attachments/assets/1f7b9e4d-1efa-4d61-bfa1-9a9e5803fcdf" />


## Walkthrough

### Step 1: Open the Provided Link

Browse to the link the room provides. It lands on a **login page** asking for a username and password — with no registration option and no credentials given in the room text.

<img width="2533" height="1069" alt="step-3" src="https://github.com/user-attachments/assets/010c89dd-ff6a-407d-9cc8-868da71e02cb" />


### Step 2: View the Source — the Credentials Are in a Comment

Before guessing anything, read the page the way an attacker does: press **Ctrl+U** to view the source. An HTML comment in the markup states plainly that visitors should log in with **guest** as both the username and the password. Developers leave notes like this in production code constantly, and the browser serves them to anyone who looks.

<img width="2470" height="1545" alt="step-4" src="https://github.com/user-attachments/assets/745c79a6-f362-4136-9140-3a1f0ca58cc0" />


### Step 3: Log In as Guest

Enter `guest` / `guest` in the form and submit:

<img width="2484" height="1119" alt="step-5" src="https://github.com/user-attachments/assets/b6ea23ac-71cd-4c0d-842a-3925437da5e1" />


The login works — a guest session opens. Note the address bar: the application addresses the page by a **`user` parameter** whose value is `guest`.

<img width="2855" height="779" alt="step-6" src="https://github.com/user-attachments/assets/e3990f44-39c2-4990-929a-f6516d57526b" />


### Step 4: Change `guest` to `admin` — Flag

Edit the parameter in the URL, replacing `guest` with **`admin`**, and load the page. No password is requested and no check intervenes: the server builds the page for whichever user the URL names, and the admin's page contains the **flag** (value in the screenshot, not repeated here).

<img width="2778" height="788" alt="step-7" src="https://github.com/user-attachments/assets/0f935592-7fac-4d1e-8e0f-5eb649490881" />


## Key Takeaways

- Always view the source. An HTML comment handed over working credentials — comments, like error messages and metadata, are part of the attack surface and ship to every visitor.
- Identity taken from a client-controlled parameter is not authentication. The session said "guest"; the URL said "admin"; the server believed the URL. Authorisation must be derived from the server-side session on every request.
- This is the same family as IDOR and Corridor's hashed rooms: the reference changed, the missing check identical. When any part of a request names a user or an object, change it and see what the server does.
- Demo and guest accounts are real accounts: if a guest login exists, its privileges — and its ability to pivot into parameter tampering — must be designed, not assumed away.

## Remediation

Strip comments and developer notes from production markup, and never document credentials in client-side code. Derive the caller's identity exclusively from the server-side session, ignore any user identifier supplied in URLs or form fields, and enforce per-object authorisation checks on every request. Disable or strictly limit guest accounts in production, and log and alert on parameter values that do not match the authenticated session — that mismatch is the attack, happening in plain sight.
