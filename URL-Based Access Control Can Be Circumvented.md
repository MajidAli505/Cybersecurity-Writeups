# URL-Based Access Control Can Be Circumvented

**Platform:** PortSwigger Web Security Academy
**Category:** Access Control
**Difficulty:** Practitioner
**Status:** ✅ Solved

## Overview

Layered architectures often enforce access rules in a front-end system — a proxy or load balancer that blocks requests to sensitive paths before they ever reach the application. That only works if the front-end and the back-end agree on what URL was requested. Many back-end frameworks support override headers such as `X-Original-URL`, originally meant to remember the original URL when a proxy rewrites a request. If the back-end routes on the header while the front-end checks the real request line, the two layers can be made to see two different URLs at once — and the block evaporates.

## Lab Objective

This website has an unauthenticated admin panel at `/admin`, but a front-end system blocks external access to that path. The back-end application is built on a framework that supports the `X-Original-URL` header. Access the admin panel and delete the user `carlos`.

<img width="2185" height="947" alt="step-1" src="https://github.com/user-attachments/assets/922e725b-67c9-4d1a-880a-d74f51a9aff2" />


## Walkthrough

### Step 1: Request /admin and Read the Block

Try to load `/admin` and observe that you get blocked. Notice that the response is very plain, suggesting it may originate from a front-end system rather than the application itself — the application never saw this request.

<img width="3774" height="1520" alt="step-2" src="https://github.com/user-attachments/assets/ab7bdd6e-ef2c-4c09-9357-dc20cca3c217" />


<img width="1976" height="657" alt="step-3" src="https://github.com/user-attachments/assets/b472f962-9447-4f3b-a915-0e741133c872" />


### Step 2: Probe the Back-End with X-Original-URL

Send the request to Burp Repeater. Change the URL in the request line to `/` and add the HTTP header `X-Original-URL: /invalid`:

```http
GET / HTTP/2
Host: <lab host>
X-Original-URL: /invalid
```

Observe that the application returns a "not found" response. The front-end let the request through because the real path is just `/` — but the back-end answered for `/invalid`, a path that was never in the request line. This indicates that the back-end system is processing the URL from the `X-Original-URL` header.

<img width="2563" height="1228" alt="step-4" src="https://github.com/user-attachments/assets/223afc94-b122-40da-8f6f-1564f63c0edb" />


<img width="2367" height="1276" alt="step-5" src="https://github.com/user-attachments/assets/704c2406-1606-4c95-963f-935d360b215e" />


### Step 3: Open the Admin Panel Through the Header

Change the value of the `X-Original-URL` header to `/admin`, keeping the request line at `/`:

```http
GET / HTTP/2
Host: <lab host>
X-Original-URL: /admin
```

Observe that you can now access the admin page. The front-end checked one URL; the back-end served another.

<img width="2553" height="1240" alt="step-6" src="https://github.com/user-attachments/assets/855f8247-4119-46b1-a363-99ddfa3205a6" />


### Step 4: Delete carlos

Admin actions live under the same protected path, so they can be reached the same way. To delete `carlos`, add `?username=carlos` to the real query string — the back-end reads its parameters from there — and change the `X-Original-URL` path to `/admin/delete`:

```http
GET /?username=carlos HTTP/2
Host: <lab host>
X-Original-URL: /admin/delete
```

Send the request. The user is deleted and the lab is solved.

<img width="2501" height="1259" alt="step-7" src="https://github.com/user-attachments/assets/3fc268b4-1062-444b-9559-728e88fe44a3" />


<img width="3280" height="1565" alt="step-8" src="https://github.com/user-attachments/assets/01ed8737-c259-4747-8356-1fcc1048b111" />


## Key Takeaways

- A block is only as strong as the agreement between the layers enforcing and serving it: if the front-end and back-end can be made to disagree about the requested URL, the control fails open.
- Headers like `X-Original-URL` and `X-Rewrite-URL` exist for trusted proxy rewrites. Honoring them from untrusted clients hands those clients the routing table.
- The tell was in the error: a bare, unstyled block response pointed at a separate front-end system, and a "not found" for a path that only existed in a header confirmed the split.

## Remediation

Enforce access control at the back-end, on the same URL the application actually routes — never rely solely on a front-end path block. Frameworks should ignore `X-Original-URL` and similar override headers unless they come from a trusted internal proxy, and proxies should strip these headers from external requests. Every layer that makes a decision about a request should base it on the one canonical URL the request will be served under.
