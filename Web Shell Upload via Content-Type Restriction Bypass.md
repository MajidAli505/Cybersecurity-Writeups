# Web Shell Upload via Content-Type Restriction Bypass

**Platform:** PortSwigger Web Security Academy
**Category:** File Upload
**Difficulty:** Apprentice
**Status:** ✅ Solved

## Background

In a multipart upload, every file part carries a `Content-Type` header that the *client* supplies — it is a claim, not a measurement. An application that decides "is this an image?" by reading that header is asking the uploader to grade their own homework. The fix-free version of the check looks convincing from the UI (real rejections happen), but anyone with a proxy can keep the malicious file and simply rewrite its declared type to an allowed one. The server, meanwhile, still stores and executes the file according to its `.php` extension.

## Lab Objective

The image upload function tries to block unexpected file types, but its check relies on user-controllable input. Upload a basic PHP web shell, read `/home/carlos/secret` with it, and submit the secret via the lab banner.

**Provided resources:** your account `wiener:peter`.

<img width="2149" height="1159" alt="step-1" src="https://github.com/user-attachments/assets/a1d42eb5-57a9-41c1-8f3b-0d88f96a8fe0" />

## Walkthrough

### Step 1: Log In and Try the Shell Directly

Log in and open the avatar upload on your account page. Prepare the same one-line PHP shell that reads the target file, and first try uploading it as-is: the application rejects it, stating that only PNG or JPG images are allowed. So a check exists — the question is what it actually inspects.

<img width="2602" height="1365" alt="step-2" src="https://github.com/user-attachments/assets/9283d088-e8ba-4493-9aef-b31a452d00ac" />

<img width="2516" height="1170" alt="step-3" src="https://github.com/user-attachments/assets/c7101cbc-a832-4e9e-98e5-067fac7d59e2" />

<img width="2860" height="1445" alt="step-4" src="https://github.com/user-attachments/assets/e0b43021-a164-4cfd-be30-659a24984b1e" />

<img width="2106" height="1386" alt="step-5" src="https://github.com/user-attachments/assets/59bbc8c2-1ac6-4deb-8720-684a21608111" />

<img width="2620" height="330" alt="step-6" src="https://github.com/user-attachments/assets/dee316f3-6526-4d81-ba5c-1a6c8954e643" />

### Step 2: Inspect the Upload Request in Burp

Repeat the upload with Burp intercepting. The multipart request shows the file part's metadata: the filename ends in `.php` and the part declares its own `Content-Type`. The rejection keys off that declared type — a value *you* control in the request.

<img width="2945" height="1129" alt="step-7" src="https://github.com/user-attachments/assets/48079e69-5eda-4185-a73e-46b1585a8bd7" />

### Step 3: Lie About the Content-Type

While the request is intercepted, change nothing about the file itself — same `.php` filename, same PHP code — but rewrite the part's declared type to an allowed image type:

```http
Content-Disposition: form-data; name="avatar"; filename="shell.php"
Content-Type: image/png
```

Forward the request. This time the upload is accepted: the check believed the header. And because the stored file keeps its `.php` name in the web-accessible avatars directory, requesting its URL executes it — the response prints the contents of `/home/carlos/secret`. Submit the secret via the lab banner. The lab is solved.

<img width="1539" height="1496" alt="step-8" src="https://github.com/user-attachments/assets/ace41924-810f-4f9f-aae0-e29f5cb63af9" />

<img width="2650" height="1206" alt="step-9" src="https://github.com/user-attachments/assets/d12af371-ad84-4cf7-8be6-2632fb5772f1" />

<img width="2936" height="1239" alt="step-10" src="https://github.com/user-attachments/assets/0aab4fb4-2f43-465d-95fd-674e14576024" />

<img width="2766" height="778" alt="step-11" src="https://github.com/user-attachments/assets/12cbe6b7-395b-43d9-a2c2-e2bb4ca4bb3a" />

<img width="3091" height="1171" alt="step-12" src="https://github.com/user-attachments/assets/60aa1162-c53e-4eee-aa90-6edf5453e066" />

## Key Takeaways

- A `Content-Type` header in an upload is attacker-controlled metadata. Any security decision based on it alone is decorative.
- Compare what rejected the request with what the server later trusts: here the *check* used the header, but *execution* used the file extension — the mismatch is the vulnerability.
- Intercepting one failed upload tells you exactly which field the validator reads; that is always the first diagnostic step when an upload is refused.

## Remediation

Determine file type from the file's actual bytes (magic-byte/content inspection) on the server, enforce an extension allowlist, and store uploads with generated names outside executable locations. Client-supplied headers and filenames should never influence either the validation outcome or how the stored file is later handled.
