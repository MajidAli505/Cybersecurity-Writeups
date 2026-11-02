# Cybersecurity Writeups

**Step-by-step penetration testing writeups for PortSwigger Web Security Academy, TryHackMe, and Hack The Box.**

Every writeup documents a lab I solved myself: the vulnerability, the full exploitation process with screenshots at each step, the evidence, and why the flaw exists. Written in professional English so anyone can reproduce the attack and learn from it.

![Labs](https://img.shields.io/badge/PortSwigger%20labs%20published-39%2B-brightgreen)
![Platforms](https://img.shields.io/badge/platforms-PortSwigger%20%C2%B7%20TryHackMe%20%C2%B7%20Hack%20The%20Box-blue)

---

## PortSwigger Web Security Academy

### ✅ Authentication — 11 labs
- Username Enumeration via Response Timing
- Username Enumeration via Account Lock
- Broken Brute-Force Protection — Multiple Credentials per Request
- Bypassing Two-Factor Authentication
- Brute-Forcing a Stay-Logged-In Cookie
- Offline Password Cracking
- Password Reset Broken Logic
- Password Reset Poisoning via Middleware
- Password Brute-Force via Password Change
- Basic Password Reset Poisoning
- Password Reset Poisoning via Dangling Markup

### ✅ Information Disclosure — 5 labs
- Information Disclosure in Error Messages
- Information Disclosure on Debug Page
- Source Code Disclosure via Backup Files
- Authentication Bypass via Information Disclosure
- Information Disclosure in Version Control History

### ✅ Access Control — 13 labs
- Unprotected Admin Functionality
- Unprotected Admin Functionality with Unpredictable URL
- User Role Controlled by Request Parameter
- User Role Can Be Modified in User Profile
- URL-Based Access Control Can Be Circumvented
- Method-Based Access Control Can Be Circumvented
- User ID Controlled by Request Parameter
- User ID Controlled by Request Parameter with Unpredictable User IDs
- User ID Controlled by Request Parameter with Data Leakage in Redirect
- User ID Controlled by Request Parameter with Password Disclosure
- Insecure Direct Object References (IDOR)
- Multi-Step Process with No Access Control on One Step
- Referer-Based Access Control

### ✅ CSRF
- CSRF with Broken Referer Validation
- Bypassing Referer-Based CSRF Defenses
- Bypassing SameSite Lax Restrictions with Newly Issued Cookies
- Bypassing SameSite Restrictions via a Vulnerable Sibling Domain
- Bypassing SameSite Restrictions Using On-Site Gadgets

### 🔄 Business Logic — in progress
- Excessive Trust in Client-Side Controls
- High-Level Logic Vulnerability
- Low-Level Logic Flaw
- Inconsistent Handling of Exceptional Input
- Inconsistent Security Controls
- …more on the way

### Also here
- Cross-Site WebSocket Hijacking (WebSockets)
- Metasploit: Post-Exploitation (TryHackMe)

**Coming next:** OS Command Injection · API Testing · Path Traversal · File Upload · Web Cache Deception · XXE · NoSQL Injection · XSS · CORS

---

## How each writeup is structured

1. **Lab objective** — what the lab asks and what a successful exploit looks like
2. **Reconnaissance** — exploring the application and spotting the flaw
3. **Exploitation** — the attack, step by step, mostly in Burp Suite, with screenshots
4. **Result** — evidence the lab is solved and the root cause of the vulnerability

## About the author

**Majid Ali** — cybersecurity student and aspiring penetration tester, focused on web application security and Active Directory attacks.

- 🐺 Check out my tool: [AutoHound](https://github.com/MajidAli505/AutoHound) — automatic Active Directory attack-path analysis from SharpHound data
- 👤 Profile: [@MajidAli505](https://github.com/MajidAli505)

> All labs are solved in legal, purpose-built training environments. This repository is for education and professional development.
