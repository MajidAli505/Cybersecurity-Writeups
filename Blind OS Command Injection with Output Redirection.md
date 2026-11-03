# Blind OS Command Injection with Output Redirection

**Platform:** PortSwigger Web Security Academy
**Category:** OS Command Injection
**Difficulty:** Practitioner
**Status:** ✅ Solved

## Background

When command output never reaches the HTTP response, the shell itself can be turned into the delivery channel: redirect the injected command's output into a file inside a web-served directory, then simply request that file through the application. The one requirement is a folder that is both writable by the server user and readable over HTTP. This lab provides exactly that — the product images directory — so a blind injection becomes a readable one in two requests.

## Lab Objective

The feedback function contains a blind OS command injection flaw: user-supplied details end up in a shell command whose output is not returned. The folder `/var/www/images/` is writable, and the application serves the product catalog's images from it. Run `whoami`, redirect its output to a file in that folder, and retrieve the file through the image-loading URL.

<img width="2181" height="1504" alt="step-1" src="https://github.com/user-attachments/assets/10e94bee-71b2-43b7-9d8d-68b2e8129417" />

## Walkthrough

### Step 1: Submit Feedback and Capture the Request

Open the feedback page, submit the form once, and send the captured request to Burp Repeater.

<img width="3127" height="1775" alt="step-2" src="https://github.com/user-attachments/assets/48d00ad4-1339-445d-ab5e-d18de5f03b04" />

<img width="3010" height="1629" alt="step-3" src="https://github.com/user-attachments/assets/ee9b66dd-cb9e-4089-9898-64da0f55c76d" />

<img width="2628" height="1065" alt="step-4" src="https://github.com/user-attachments/assets/fdde4351-d9a9-4b67-900f-c48b5615a4ec" />

### Step 2: Redirect `whoami` Output into the Web Root

Modify the `email` parameter so the injected command writes its output into the images folder:

```text
email=||whoami>/var/www/images/output.txt||
```

The response looks no different from a normal submission — but on the server, a new file `output.txt` now exists in `/var/www/images/`, containing the output of `whoami`.

<img width="2887" height="1221" alt="step-5" src="https://github.com/user-attachments/assets/4ee2b7a6-b7dd-4b05-8ec2-8b2de889668b" />

### Step 3: Read the File Back Through the Image Loader

Now find the request the application makes when it loads a product image, and send it to Repeater. Its `filename` parameter selects the file to serve. Change it to the file you just created:

```text
filename=output.txt
```

The response body is the file's content — the username printed by `whoami`. The blind injection has been converted into a readable one, and the lab is solved.

<img width="2629" height="1020" alt="step-6" src="https://github.com/user-attachments/assets/78eb57c6-c138-4607-bdcf-8d9f62be2861" />

<img width="2475" height="884" alt="step-7" src="https://github.com/user-attachments/assets/8da6217f-0cd8-4263-86d8-073dbe1ef194" />

<img width="3020" height="1278" alt="step-8" src="https://github.com/user-attachments/assets/defef8e3-e18f-450a-ad0a-4b38408e38a4" />

## Key Takeaways

- Output redirection upgrades blind injection to visible injection whenever a writable, web-accessible directory exists. The web root and asset folders (images, uploads, exports) are the usual candidates.
- Two chained primitives did the work: the injection (write) plus an existing file-serving feature (read). individually each looks minor; together they leak command output.
- Nothing about the feedback response hinted at success — confirming the write required reading the file back, so always verify through the second channel rather than assuming.

## Remediation

Never construct shell commands from user input; invoke programs with argument arrays or use native APIs, and validate inputs strictly (an email must parse as an email). Additionally, the web server should not serve arbitrary files from writable directories: keep upload/image folders outside direct web access or serve them through a handler that enforces an allowlist of names and types, and run the application with the least file-system privileges it needs.
