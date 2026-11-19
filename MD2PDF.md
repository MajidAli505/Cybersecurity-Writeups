# MD2PDF

**Platform:** TryHackMe
**Category:** Challenge Room
**Difficulty:** Easy
**Status:** ✅ Solved (flag captured)

## Background

MD2PDF converts Markdown to PDF in the browser's favourite way: you paste markup, a server-side renderer turns it into a document. Anything a server *renders* is a request the server makes — and that is **Server-Side Request Forgery (SSRF)** waiting for an address. Directory fuzzing finds an `/admin` page that refuses visitors: the real admin interface listens on **`localhost:5000`**, reachable only from the machine itself. But the PDF converter *is* the machine. Embed an `<iframe src="http://localhost:5000/admin">` in the Markdown, convert, and the renderer fetches the internal page from inside the perimeter and prints it into the PDF — flag included. The room's lesson is that "internal only" means nothing when a public feature can be told to fetch internal URLs.

## Lab Objective

Find the internal admin service on `localhost:5000`, then use the Markdown-to-PDF converter's rendering to SSRF it and read the flag from the generated PDF.

**Provided resources:** the target machine only; no credentials are given.

<img width="3054" height="974" alt="step-1" src="https://github.com/user-attachments/assets/34fde70a-4a2b-4014-a90c-b6e4b069c422" />


<img width="2766" height="780" alt="step-2" src="https://github.com/user-attachments/assets/466c11d1-7bda-44b0-98fb-3e845ef0d4c8" />


## Walkthrough

### Step 1: Browse the Converter

Open the target IP. The page is the MD2PDF tool itself: a Markdown input and a convert button. On its own it offers no clue about where the flag lives, so enumerate.

<img width="2729" height="1286" alt="step-3" src="https://github.com/user-attachments/assets/ce2ccb9a-e426-4d9d-9354-b1f27489be7c" />


### Step 2: Fuzz Directories — `/admin`

Run ffuf against the site:

```bash
ffuf -u http://10.49.x.x/FUZZ -w /usr/share/wordlists/dirb/common.txt
```

One directory stands out: **`/admin`**.

<img width="2801" height="1087" alt="step-4" src="https://github.com/user-attachments/assets/93d0dc1e-0723-4cb8-a964-518cfec85993" />


### Step 3: The Admin Page Points Inward

Visit `/admin`. It does not open the panel — instead it explains, in effect, that the admin interface is only accessible **from localhost (127.0.0.1), on port 5000**. That is the target and the obstacle in one message: a service exists at `http://localhost:5000/admin`, and no external request can reach it. What is needed is a feature on this server that makes requests *for* you.

<img width="2574" height="752" alt="step-5" src="https://github.com/user-attachments/assets/e0ff9f03-5a66-4fea-8268-677ca969d668" />


<img width="1820" height="557" alt="step-6" src="https://github.com/user-attachments/assets/83c48dac-4afd-4172-bf4d-a49d8547fcfb" />


### Step 4: SSRF Through the PDF Renderer

The converter is exactly that feature. HTML embedded in Markdown survives into the rendered document, and an iframe makes the renderer fetch its source. Submit Markdown containing:

```html
<iframe src="http://localhost:5000/admin"></iframe>
```

and convert it to PDF. The server-side renderer resolves `localhost` *as itself*, loads the internal admin page, and embeds the result in the document.

### Step 5: Read the Flag from the PDF

Open the generated PDF: the internal admin page is rendered inside it — and with it, the **flag** (value in the screenshot, not repeated here). The perimeter check was never bypassed by force; it was bypassed by asking the insider to fetch the page.

<img width="1709" height="758" alt="step-7" src="https://github.com/user-attachments/assets/18470a4f-5a11-4586-9949-61be4fb11655" />


## Key Takeaways

- Every server-side renderer, converter, previewer and URL fetcher is an SSRF candidate. If a feature makes the server request a URL you influence, you choose where the server goes — including `localhost`.
- "Accessible only from 127.0.0.1" is a network assumption, not authentication. One SSRF anywhere on the box collapses it, because the request genuinely does come from localhost.
- Enumeration found the target: without ffuf's `/admin` hit and its helpful message, there would be no address to aim the iframe at. Content discovery and SSRF are two halves of one exploit.
- HTML injection into a Markdown converter is not just XSS — in a server-side rendering pipeline it is a request forgery toolkit (iframes, images, CSS imports all fetch).

## Remediation

Renderers and fetchers must never request arbitrary URLs: allow-list destinations, block loopback, link-local and private ranges at the fetch layer (and re-check after redirects and DNS resolution, to defeat rebinding). Sanitise Markdown to plain markup — strip raw HTML, especially iframes and other fetching elements — before rendering. Internal admin services should require authentication in their own right, not trust the source IP, and should bind behind a separate internal interface with firewall rules that application servers cannot cross. Finally, run converters in a sandbox with no network egress to internal ranges.
