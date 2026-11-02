# Information Disclosure in Error Messages

**Platform:** PortSwigger Web Security Academy
**Category:** Information Disclosure
**Difficulty:** Apprentice
**Status:** ✅ Solved

## Overview

Applications that respond to invalid input with detailed, developer-oriented error messages hand attackers a free reconnaissance tool. A single malformed request can make the server spill its stack trace — revealing the programming language, internal code structure, and, most valuably, the exact versions of the frameworks it runs on. Knowing an exact framework version lets an attacker look up known vulnerabilities for that release instead of probing blindly.

## Lab Objective

Trigger a verbose error message, identify the third-party framework and its version from what the error reveals, and submit the version number as the solution.

<img width="2240" height="780" alt="step-1" src="https://github.com/user-attachments/assets/d9dbdd60-4e73-477e-837d-5158d42e076d" />

## Walkthrough

### Step 1: Break the productId Parameter

Browse to a product page and tamper with the `productId` parameter in the URL — here, appending a single quote so the value becomes `3'`:

```text
/product?productId=3'
```

The application expects an integer, so this malformed value is a reliable way to force an exception.

<img width="2728" height="545" alt="step-2" src="https://github.com/user-attachments/assets/940b9c03-cf1f-42c3-a867-12f2c3f03b58" />

### Step 2: Read the Verbose Error

Instead of a generic error page, the server returns a full internal error: a Java `NumberFormatException` ("For input string: "3'"") followed by the complete stack trace, exposing internal class and method names. At the very bottom of the error output, the framework identifies itself:

```text
Apache Struts 2 2.3.31
```

<img width="2345" height="1587" alt="step-3" src="https://github.com/user-attachments/assets/64fbe1ad-c508-458e-9718-26c6b06307b7" />

### Step 3: Submit the Framework Version

Click **Submit solution** and enter the framework name and version exactly as disclosed by the error message:

```text
Apache Struts 2 2.3.31
```

<img width="3246" height="1203" alt="step-4" src="https://github.com/user-attachments/assets/bd6d344e-1d3a-48f6-b42d-5eb05948e7c3" />

### Step 4: Lab Solved

The submission is accepted and the lab is complete — one stray character in a URL was all it took to fingerprint the application's stack.

<img width="3333" height="1121" alt="step-5" src="https://github.com/user-attachments/assets/8309d0fc-1c22-4974-9d7c-4149ae1ce2e6" />

## Key Takeaways

- Verbose errors are reconnaissance gold: stack traces disclose languages, frameworks, file paths, and versions that map directly to public vulnerability databases.
- Triggering errors is trivial — unexpected characters, wrong data types, and missing parameters in any input field are worth testing early in an assessment.
- The disclosed version here (an old Struts 2 release) is precisely the kind of detail that turns a generic target into a specific, known-exploitable one.

## Remediation

- Return generic, user-friendly error messages in production and log the detailed diagnostics server-side only.
- Disable framework debug and stack-trace output in production configuration.
- Keep third-party frameworks patched, and avoid advertising version numbers in responses, headers, or error pages.
