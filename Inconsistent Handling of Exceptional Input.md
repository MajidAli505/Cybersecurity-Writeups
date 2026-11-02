# Inconsistent Handling of Exceptional Input

**Platform:** PortSwigger Web Security Academy
**Category:** Business Logic
**Difficulty:** Practitioner
**Status:** ✅ Solved

## Background

Applications often process the same input in more than one place, and each place may handle extremes differently. In this lab, two components disagree about long email addresses: the mail system happily sends a confirmation message to a very long address, but the account database stores the address in a field capped at **255 characters** and silently truncates anything longer. Neither behavior is a vulnerability on its own. Together, they let an attacker register with an address that is *delivered* as one domain but *stored* as another — and the stored domain is what decides who gets admin access.

## Lab Objective

This lab doesn't adequately validate user input. Exploit a logic flaw in its account registration process to gain access to administrative functionality, then delete the user `carlos`.

<img width="2718" height="1225" alt="step-1" src="https://github.com/user-attachments/assets/571b2d95-b353-4fd2-be47-2f9afb6a362e" />

## Walkthrough

### Step 1: Discover the Admin Panel and Its Rule

With traffic proxying through Burp, run content discovery against the site (Target → Site map → right-click the domain → Engagement tools → Discover content). Among the results is a path the interface never links to: `/admin`.

Browse to it. You are refused — but the error message is informative: access is restricted to users from **DontWannaCry**. The registration page reinforces this, telling DontWannaCry employees to register with their company email address. So admin access follows from owning an `@dontwannacry.com` address — an address you obviously cannot receive mail at. Unless the system can be made to think you have one.

<img width="3186" height="1665" alt="step-2" src="https://github.com/user-attachments/assets/e831a43a-f8ef-4d43-ac2e-f0b5279c1f65" />

<img width="2789" height="1309" alt="step-3" src="https://github.com/user-attachments/assets/d12064bd-01eb-4062-b7e9-23ddc907271d" />

<img width="2315" height="1252" alt="step-4" src="https://github.com/user-attachments/assets/d81a76dc-ab60-4c7e-b7bd-c66479460c90" />

<img width="2017" height="1066" alt="step-5" src="https://github.com/user-attachments/assets/781a4b75-3d6b-4668-a4ee-8a26a08cae90" />

### Step 2: Set Up a Controllable Mailbox

Open the email client from the lab banner (the exploit server). Your mailbox lives on a domain containing a unique ID, in the form `@<unique-id>.exploit-server.net`. Note it down — every address under that domain delivers to you, including addresses with extra subdomains in front of it.

<img width="2991" height="1498" alt="step-6" src="https://github.com/user-attachments/assets/9df4b45a-dd19-4ee5-aeee-1f1d9439fda6" />

<img width="2395" height="1367" alt="step-7" src="https://github.com/user-attachments/assets/1741dd1c-15fb-4378-871f-b1b997cda38e" />

### Step 3: Probe the Truncation with a Very Long Address

Register a throwaway account using an exceptionally long email address at your exploit-server domain — a local part of 200+ characters. Open the confirmation email in the client and click the link to finish registering; delivery works fine with the long address.

Now log in and open "My account". The surprise: the address shown for the account has been cut off at exactly **255 characters**. The application sent mail using the full address, but stored a truncated one. That inconsistency is the vulnerability.

<img width="3067" height="1635" alt="step-8" src="https://github.com/user-attachments/assets/ee81d0b2-74ad-43b9-8e2f-b7f9d621398e" />

<img width="3067" height="1725" alt="step-9" src="https://github.com/user-attachments/assets/094d0fdf-d1dd-4187-bc9c-28b7e8a8b085" />

<img width="3213" height="1121" alt="step-10" src="https://github.com/user-attachments/assets/90f0fcc3-5ceb-45e1-9a23-2ac794650b3b" />

<img width="3067" height="1433" alt="step-11" src="https://github.com/user-attachments/assets/f6684edd-694f-428b-a360-391017e15384" />

### Step 4: Craft an Address That Truncates to the Target Domain

Log out and register again, this time with the target domain smuggled in as a subdomain:

```text
<very-long-string>@dontwannacry.com.<your-unique-id>.exploit-server.net
```

Size the long string precisely so that the **final "m" of `dontwannacry.com` lands exactly on character 255**. Above that length, everything — `.exploit-server.net` included — will be cut off by the truncation. The address still routes mail to your exploit server (DNS delivers on the full domain), so the confirmation email still reaches you.

<img width="3067" height="1498" alt="step-12" src="https://github.com/user-attachments/assets/187ec15e-9751-4365-9115-e5d6c6a2e2f1" />

<img width="2950" height="1587" alt="step-13" src="https://github.com/user-attachments/assets/eb781398-370e-47cb-be3d-39466a1f70a1" />

<img width="2180" height="1145" alt="step-14" src="https://github.com/user-attachments/assets/75103018-2d25-40cf-b05d-500d4cce1dca" />

### Step 5: Confirm, Log In, Delete carlos

Click the confirmation link in the email and log in with the new account. On "My account", the stored address now ends in `@dontwannacry.com` — as far as the application is concerned, you are a DontWannaCry employee. The admin panel is accessible. Open it and delete `carlos`. The lab is solved.

<img width="3067" height="1813" alt="step-15" src="https://github.com/user-attachments/assets/51258dfb-7faf-44eb-89bf-be19483c555a" />

<img width="3067" height="703" alt="step-16" src="https://github.com/user-attachments/assets/7b3a8f19-1e44-44b8-a051-e17388a61043" />

<img width="3213" height="1090" alt="step-17" src="https://github.com/user-attachments/assets/6ca6fc8e-a43d-4ba7-a8c8-597cae509abd" />

<img width="3117" height="987" alt="step-18" src="https://github.com/user-attachments/assets/92ba8dbb-6afa-4de6-95ac-d80da3e5a62b" />

<img width="3676" height="1272" alt="step-19" src="https://github.com/user-attachments/assets/70e03946-2831-4dd2-9316-36203d772f9a" />

## Key Takeaways

- Truncation is a silent transformation: the system that *sends* and the system that *stores* ended up working with two different addresses, and every downstream decision used the stored one.
- Access decisions based on an email domain are only as strong as the registration path's ability to prove control of that domain — here the proof (the confirmation email) was checked against the un-truncated address while the privilege was granted on the truncated one.
- "Exceptional" input — overlong strings, boundary lengths like exactly 255 — is where inconsistent handling hides. Test at and around every limit you can find.

## Remediation

Handle input consistently across the whole pipeline: validate length limits *before* processing, and reject (don't silently truncate) anything that exceeds them — especially identity attributes like email addresses. Any security decision based on an email domain must be made against the exact address whose ownership was verified, and the verified value and the stored value must be identical by construction.
