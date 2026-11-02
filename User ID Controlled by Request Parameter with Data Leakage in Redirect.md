# User ID Controlled by Request Parameter with Data Leakage in Redirect

**Platform:** PortSwigger Web Security Academy
**Category:** Access Control
**Difficulty:** Apprentice
**Status:** ✅ Solved

## Background

In some cases, an application *does* detect that the user is not permitted to access the requested resource, and responds with a redirect to the login page. However, the response carrying that redirect may still include sensitive data belonging to the targeted user — the application builds the full page, then changes its mind about sending you there. A browser obediently follows the redirect and never shows the body, so the leak is invisible in normal use. Anyone watching the raw response, though, receives everything. The attack still succeeds.

## Lab Objective

Obtain the API key for the user `carlos` and submit it as the solution.

**Provided resources:** your credentials `wiener:peter`.

## Walkthrough

### Step 1: Log In and Open Your Account Page

Log in using the supplied credentials and open your account page. As in the other labs in this series, the account is selected by a parameter in the URL: `/my-account?id=wiener`, and the page displays your API key.


<img width="3130" height="1645" alt="step-1" src="https://github.com/user-attachments/assets/48c108fc-a7a4-4eed-9c18-1efbfadd7c0f" />

<img width="2747" height="1454" alt="step-2" src="https://github.com/user-attachments/assets/84b2c7b9-546f-4ba5-8fd7-6da495437978" />

<img width="2826" height="1583" alt="step-3" src="https://github.com/user-attachments/assets/c09c0660-99e9-46f0-8891-b829d894541e" />

### Step 2: Change the ID and Get Bounced

Change the ID parameter to `carlos` and press Enter. This time you do not see his account page — the application detects the mismatch and sends you back to the login page. It looks as though the access control works.

<img width="2414" height="1541" alt="step-4" src="https://github.com/user-attachments/assets/a229869a-0b7a-4da8-b97d-6e3ffabf825f" />

<img width="2500" height="1525" alt="step-5" src="https://github.com/user-attachments/assets/cbc21651-03ef-4018-9ffa-b3ee3757b08d" />

### Step 3: Read the Redirect Response in Burp

Repeat the request with Burp capturing the traffic, and inspect the response to `GET /my-account?id=carlos` in the proxy history. The response is a redirect to the login page — but its body still contains the fully rendered account page for `carlos`, including the line "Your API Key is:" followed by his key. The server generated the sensitive content first and attached the redirect afterwards; the browser hides this, Burp does not.

<img width="2989" height="1605" alt="step-6" src="https://github.com/user-attachments/assets/23238b3e-492f-4ff5-b0e1-ba4dbdc181ce" />

### Step 4: Submit the API Key

Copy the API key from the response body and submit it using the lab's "Submit solution" button. The lab is solved.

<img width="3005" height="1471" alt="step-7" src="https://github.com/user-attachments/assets/3a679134-3be5-4ae0-80a7-bc89d5a84c65" />

<img width="3249" height="1497" alt="step-8" src="https://github.com/user-attachments/assets/a622b890-c654-4475-9879-58cc12fad9c5" />

## Key Takeaways

- A redirect is not a refusal. If the sensitive content is in the response body, the data has already been disclosed — what the browser chooses to display afterwards is irrelevant.
- This is why proxy tools matter: vulnerabilities that hide behind redirects, and other client-side behavior, are only visible at the request/response level.
- The underlying flaw is ordering: the application loaded and rendered another user's data before (or regardless of) completing the authorization decision.

## Remediation

Make the authorization decision before any data is loaded or rendered, and terminate processing when it fails — a redirect response should have an empty body, not a fully built page. Structure request handling so that "not permitted" exits the handler entirely rather than swapping the status and headers on an already-generated response.
