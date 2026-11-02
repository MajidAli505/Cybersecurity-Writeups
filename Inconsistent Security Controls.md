# Inconsistent Security Controls

**Platform:** PortSwigger Web Security Academy
**Category:** Business Logic
**Difficulty:** Apprentice
**Status:** ✅ Solved

## Background

Security controls only work if they are applied everywhere the protected property can change. This application decides who is an employee by looking at the domain of the account's email address — and it carefully verifies that domain **at registration**, where the user must confirm ownership through a link sent to the address. But it applies no such control to the **change-email** feature: after logging in, you can replace your address with anything at all, unverified. A property that is verified at the front door and freely editable through the back door is not verified at all.

## Lab Objective

This lab's flawed logic allows arbitrary users to access administrative functionality that should only be available to company employees. Access the admin panel and delete the user `carlos`.

<img width="2184" height="850" alt="step-1" src="https://github.com/user-attachments/assets/442a2f5e-23ec-47b6-9d41-14102e3e6f95" />


## Walkthrough

### Step 1: Register a Legitimate Account

Open the registration form. For the email address, first open the email client from the lab banner and copy your exploit-server domain — the address you register with must be one whose mailbox you control, so the confirmation can reach you. Fill in the form with an address at that domain and register.

<img width="3080" height="1777" alt="step-2" src="https://github.com/user-attachments/assets/d52e94a5-7e20-4cfb-8400-96db371b532a" />


<img width="3007" height="1621" alt="step-3" src="https://github.com/user-attachments/assets/dcd59ab9-47b8-4b4f-99d7-1d44847ef6c7" />


<img width="2220" height="1100" alt="step-4" src="https://github.com/user-attachments/assets/651b7b18-3b5f-42f5-8ec3-b41b7d487188" />


<img width="2328" height="1716" alt="step-5" src="https://github.com/user-attachments/assets/9a1e5468-73b7-424e-aee1-312a0b6176c8" />


### Step 2: Confirm via Email and Log In

The confirmation link arrives in the email client. Open it to complete the registration, then log in with your new account. So far, everything is legitimate: the application has verified you own the address you registered with — an ordinary outside address with no privileges.

<img width="2919" height="658" alt="step-6" src="https://github.com/user-attachments/assets/c3502f6c-e9f6-4468-ad17-c40251087b41" />


<img width="3135" height="1665" alt="step-7" src="https://github.com/user-attachments/assets/76385435-3715-46bb-bd29-acf9a2df8bed" />


<img width="2984" height="1467" alt="step-8" src="https://github.com/user-attachments/assets/51f507e2-705e-472a-85af-011a0287e63a" />


<img width="3054" height="1426" alt="step-9" src="https://github.com/user-attachments/assets/1654ecc1-99ee-4da4-8769-d495573c1bb3" />


### Step 3: Change Your Email to the Company Domain

On the "My account" page, notice the option to change your email address. Administrative access in this application is granted to `@dontwannacry.com` addresses — and the change-email form performs no verification at all: no confirmation link, no ownership check. Change your email to an arbitrary address at that domain, for example `anything@dontwannacry.com`, and save.

<img width="3054" height="1426" alt="step-10" src="https://github.com/user-attachments/assets/d46fae9f-b897-4fb3-bf8d-90a03151cd5c" />


<img width="2110" height="989" alt="step-11" src="https://github.com/user-attachments/assets/e6d8e9c6-2e51-4bee-9259-00abfa29e5d0" />


### Step 4: Use the Admin Panel

The application now treats your account as a company employee's — the **Admin panel** link appears. Open it and delete `carlos`. The lab is solved.

<img width="3136" height="1354" alt="step-12" src="https://github.com/user-attachments/assets/9bbd42b1-57f6-4671-bf3b-f0b7b8a10031" />


<img width="2995" height="1001" alt="step-13" src="https://github.com/user-attachments/assets/03cc77e3-dc94-424c-b071-96931b5c0fbc" />


<img width="3063" height="1143" alt="step-14" src="https://github.com/user-attachments/assets/781cdcbb-7463-4f0c-b38c-94b72a1803f9" />


## Key Takeaways

- Verify a property every time it is set, not just the first time. The registration check was sound; the identical value became an attack vector the moment a second, unchecked path could set it.
- Tying privileges to a mutable attribute (an email address) makes every feature that can mutate that attribute part of the security boundary — including humble account-settings forms.
- Consistency is the control: if two code paths can produce the same state, they must enforce the same rules to get there.

## Remediation

Require verification for email changes just as for registration: send a confirmation link to the *new* address and only apply the change (and any privilege derived from its domain) once ownership is proven. Better still, base administrative privileges on an explicit role granted by an administrator, not on a user-editable attribute like an email domain.
