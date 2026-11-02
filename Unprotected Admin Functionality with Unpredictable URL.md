# Unprotected Admin Functionality with Unpredictable URL

**Platform:** PortSwigger Web Security Academy
**Category:** Access Control
**Difficulty:** Apprentice
**Status:** ✅ Solved

## Overview

A step up from a guessable `/admin` path — and no safer. In this pattern the developers give the admin panel a randomized, hard-to-guess URL (something like `/administrator-panel-yb556`) and skip server-side authorization entirely, reasoning that nobody can protect what nobody can find. This is **security by obscurity**: the secret URL is doing the job a login check should do.

The problem is that URLs leak. They end up in logs, in browser histories, in referer headers — and, as here, in the application's own client-side code. Front-end JavaScript that conditionally shows an "Admin panel" link only to administrators still has to contain the URL itself, and that script is delivered to *every* visitor, admin or not. View-source defeats the whole scheme.

## Lab Objective

Discover the hidden, unguessable admin URL from the application's client-side code, open the panel, and delete the user `carlos`.

## Background: Why Obscurity Fails

The developer's thinking is that a strange URL can't be guessed, so the panel is safe. But hiding a URL is not the same as securing it: the URL only has to leak once — and client-side code that builds role-based menus is a built-in leak, because the code is sent to all users even when the link it creates is only displayed to some of them.

<img width="2145" height="1429" alt="step-1" src="https://github.com/user-attachments/assets/18173f84-9dc0-40fb-af69-763d72f24499" />

## Walkthrough

### Step 1: Open the Lab

Access the lab. Its title confirms the setup: the admin functionality is unprotected, and its URL is designed to be unpredictable.

<img width="2186" height="969" alt="step-2" src="https://github.com/user-attachments/assets/45bceab0-af7b-4874-8ae9-b2db60a0b78f" />

### Step 2: Browse the Application

Look around the shop as a normal user. There is no admin link anywhere in the interface, and no standard admin path works — the panel is hidden behind its randomized URL.

<img width="3184" height="1886" alt="step-3" src="https://github.com/user-attachments/assets/24e46537-fec1-4f7b-a2c5-387c55974303" />

### Step 3: Inspect the Page Source and JavaScript

View the source of the page and read the JavaScript that builds the user interface. Buried in the script is the code that creates the **Admin panel** link for administrator users — complete with the full, "unpredictable" admin URL it points to. The script runs for everyone; only the link's *display* is conditional. The secret is sitting in the source.

<img width="1758" height="1093" alt="step-4" src="https://github.com/user-attachments/assets/aa233f69-6d7b-4ea4-9e3e-3a8ed2388dd6" />

### Step 4: Visit the Discovered Admin URL

Browse directly to the admin URL recovered from the JavaScript. The panel opens with no authentication or role check whatsoever and shows the user administration list.

<img width="2097" height="1001" alt="step-5" src="https://github.com/user-attachments/assets/8b65ed5a-f14c-4f51-ac8c-6f7642cc082f" />

### Step 5: Delete Carlos

Use the panel to delete the user `carlos`. The panel confirms the deletion and the lab is solved.

<img width="3179" height="1118" alt="step-6" src="https://github.com/user-attachments/assets/b0b8abd5-f8e9-4323-a77f-0cf2d255f2f4" />

## Key Takeaways

- An unguessable URL is a shared secret that leaks easily — through JavaScript, logs, screenshots, and referer headers. It is not an authorization mechanism.
- Client-side role checks ("only render the link for admins") are cosmetic. Anything the browser receives — HTML, JavaScript, comments — is public information, and server-side checks are the only ones that count.
- When testing, always read the JavaScript and page source: role-gated UI code frequently names the very endpoints it pretends to hide.

## Remediation

- Enforce authentication and authorization on the server for every administrative function, whatever its URL.
- Render role-specific UI on the server so that non-admin users never receive admin links, endpoints, or related code.
- If an obscure path is used at all, treat it as a minor supplementary measure — never as the control itself.
