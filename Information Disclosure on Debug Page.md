# Information Disclosure on Debug Page

**Platform:** PortSwigger Web Security Academy  
**Category:** Information Disclosure  
**Difficulty:** Apprentice  
**Status:** ✅ Solved

## Overview

During development, applications generate rich debugging data — detailed errors, logs, and diagnostic pages — to help developers find bugs. That information is just as useful to an attacker: debug output can expose database details, internal hostnames and credentials, file paths, session variables, and even encryption keys. The failure mode is simple and common: debug features that were perfectly reasonable in a development environment are left enabled in production, where anyone can read them.

This lab contains a leftover debug page that dumps sensitive information about the application, including its environment variables.

## Lab Objective

Find the debug page, read the `SECRET_KEY` environment variable from it, and submit that value as the solution.

<img width="2233" height="799" alt="step-1" src="https://github.com/user-attachments/assets/9c113334-ecd0-40e4-addc-ec4e43ae01d5" />


## Walkthrough

### Step 1: Browse the Application

Open the lab and look around the shop front page. Nothing on the rendered page points to a debug interface — the trail is in the page source instead.

<img width="3268" height="1782" alt="step-2" src="https://github.com/user-attachments/assets/9b318317-c684-4a81-8993-59f75820745e" />


### Step 2: Inspect the Page Source

View the HTML source of a product page. Near the end of the markup there is an HTML comment containing a disabled link:

```html
<!-- <a href=/cgi-bin/phpinfo.php>Debug</a> -->
```

Commenting out a link does not remove the page it points to — it only hides the signpost, and the source code hands that signpost to anyone who looks.

<img width="2164" height="742" alt="step-3" src="https://github.com/user-attachments/assets/dc3ed808-a54e-4670-a33a-d012661cf9be" />


### Step 3: Visit the Debug Page

Browse directly to the path from the comment:

```text
/cgi-bin/phpinfo.php
```

The page loads without any authentication: it is a full `phpinfo()` dump describing the server's PHP configuration and runtime environment.

<img width="2213" height="769" alt="step-4" src="https://github.com/user-attachments/assets/1bcc6910-8171-4fc2-96c6-db0cf24471c3" />


### Step 4: Search the Dump for the Secret

Use the browser's find function to search the page for "secret". In the **Environment** table, the `SECRET_KEY` variable is listed in plaintext along with its value — alongside other environment details such as the user the service runs as and its home directory. Copy the `SECRET_KEY` value.

<img width="3076" height="1482" alt="step-5" src="https://github.com/user-attachments/assets/83e14064-1837-4e9d-8c58-9aafad234573" />


### Step 5: Submit the SECRET_KEY

Click **Submit solution** and enter the value copied from the debug page.

<img width="3049" height="890" alt="step-6" src="https://github.com/user-attachments/assets/4f8edb63-3ade-4cdc-a1b5-e9752820d2f1" />


### Step 6: Lab Solved

The submission is accepted. A page that was presumably only ever meant for the development team gave away one of the application's core secrets to an unauthenticated visitor.

<img width="3135" height="1021" alt="step-7" src="https://github.com/user-attachments/assets/b26ee9a0-039e-4aa7-b452-030b0973aa39" />


## Key Takeaways

- Debug pages and diagnostic endpoints (like `phpinfo()`) must never be reachable in production — they disclose configuration, paths, and environment variables, which often include live secrets.
- HTML comments are visible to everyone. "Hiding" functionality by commenting out its link protects nothing; always check page source during recon.
- Environment variables concentrate an application's secrets, so anything that dumps them (debug pages, error handlers, status endpoints) is a high-value target.

## Remediation

- Remove or disable debug pages, verbose errors, and diagnostic endpoints before deploying to production, and block legacy paths such as `/cgi-bin/phpinfo.php` at the server level.
- Require authentication and IP restrictions for any diagnostic tooling that is genuinely needed.
- Keep secrets in a dedicated secrets store, audit what the application prints about its environment, and strip sensitive comments from deployed HTML.
