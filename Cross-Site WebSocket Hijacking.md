# Cross-Site WebSocket Hijacking (CSWSH)

## Lab Details

| Attribute | Value |
|------------|--------|
| Platform | PortSwigger Web Security Academy |
| Category | WebSocket Security |
| Difficulty | Apprentice |
| Status | Completed |

## Overview

This lab demonstrates a Cross-Site WebSocket Hijacking (CSWSH) vulnerability where a WebSocket endpoint relies solely on session cookies for authentication.

## Objective

The objective of this lab was to:

- Discover the WebSocket endpoint
- Analyze the handshake process
- Identify missing CSRF protections
- Exploit the vulnerability
- Access sensitive information

## Tools Used

- Burp Suite
- Browser Developer Tools
- PortSwigger Web Security Academy

## Step 1: WebSocket Discovery

During analysis of the chat functionality, a WebSocket handshake was identified.

### Request

```http
GET /chat HTTP/1.1
```

### Response

```http
HTTP/1.1 101 Switching Protocols

## Step 2: Authentication Analysis

The WebSocket handshake revealed:

- Session cookie authentication
- No CSRF protection
- No Origin validation
- No unpredictable connection token

These findings indicated a potential CSWSH vulnerability.

## Step 3: Message Analysis

### Client Message

```json
{
  "message": "example message"
}


### Server Response

```json
{
  "user": "username",
  "content": "message text"
}
```

## Step 4: Exploitation

A malicious page was created to establish a WebSocket connection using the victim's authenticated session.

### Attack Flow

1. Victim logs in.
2. Victim visits attacker-controlled page.
3. Browser sends session cookie automatically.
4. WebSocket connection is established.
5. Messages are received by the attacker.

## Impact

Successful exploitation allowed:

- Access to victim messages
- Disclosure of sensitive chat history
- Extraction of confidential information
- Interaction through the victim's session

---

## Root Cause

The vulnerability existed because:

- Authentication relied solely on session cookies.
- No anti-CSRF protection was implemented.
- Origin validation was missing.

## Remediation

1. Validate the Origin header.
2. Implement anti-CSRF tokens.
3. Use connection-specific tokens.
4. Restrict trusted origins.
5. Improve session validation.

## Skills Demonstrated

- WebSocket Security Testing
- Burp Suite Analysis
- Session Management Assessment
- Cross-Site WebSocket Hijacking (CSWSH)
- Vulnerability Verification
- Security Reporting

## Result

Successfully identified and exploited a Cross-Site WebSocket Hijacking vulnerability, resulting in exposure of sensitive information and successful completion of the lab.
