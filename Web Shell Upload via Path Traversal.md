# Web Shell Upload via Path Traversal

**Platform:** PortSwigger Web Security Academy
**Category:** File Upload
**Difficulty:** Practitioner
**Status:** ✅ Solved

## Background

A common defense for upload features is to let users upload almost anything but store it in a directory where the server will *not* execute code — files there are served as inert data. That defense holds only if the file actually lands in that directory. If the application builds the storage path from the user-supplied filename without sanitizing directory separators, the filename itself becomes a path: `../` sequences walk the file out of the safe directory into a parent folder where execution *is* enabled. A naive strip of literal `../` can still be beaten when the server URL-decodes the name *after* the check — an encoded slash (`%2f`) slips past the filter and is decoded into a real separator on the way to the filesystem.

## Lab Objective

The upload function stores files in a directory configured not to execute them, but a secondary flaw lets you escape that restriction. Upload a basic PHP web shell, read `/home/carlos/secret` with it, and submit the secret via the lab banner.

**Provided resources:** your account `wiener:peter`.

<img width="2216" height="1049" alt="step-1" src="https://github.com/user-attachments/assets/5e1984b0-1dbb-4614-9093-58dc73115a83" />

## Walkthrough

### Step 1: Upload the Shell and See Why It Doesn't Run

Log in, open the avatar upload, and submit the usual one-line PHP shell. The upload is accepted — but requesting the stored file shows its source code as plain text instead of executing it. The avatars directory is deliberately configured to serve files without running them, so a shell stored there is harmless. Analyze the upload request in Burp to see exactly where the file is being saved: uploads land under `/files/avatars/`.

<img width="2851" height="1344" alt="step-2" src="https://github.com/user-attachments/assets/e7d26040-c3b2-4b8d-8526-b33af806d9de" />

<img width="2933" height="1728" alt="step-3" src="https://github.com/user-attachments/assets/85825922-2960-4328-9914-cb6caa85ab12" />

<img width="2462" height="1418" alt="step-4" src="https://github.com/user-attachments/assets/799b86e5-6cbe-4e7b-b197-2e72aa3fef0d" />

<img width="1887" height="379" alt="step-5" src="https://github.com/user-attachments/assets/72381296-fdbe-429b-b576-aace1c55d645" />

<img width="2780" height="1198" alt="step-6" src="https://github.com/user-attachments/assets/a742d8dc-4a5c-4120-bbbc-82461b550527" />

<img width="2670" height="1141" alt="step-7" src="https://github.com/user-attachments/assets/5197f4cd-fc92-471b-bf1a-8a66d62c4fbb" />

<img width="2100" height="449" alt="step-8" src="https://github.com/user-attachments/assets/ed860558-0a0a-4c88-a439-38f69533823c" />

### Step 2: Try to Walk Out of the Directory

In Repeater, replay the upload request (`POST /my-account/avatar`) with a traversal sequence prepended to the filename in the `Content-Disposition` header:

```http
Content-Disposition: form-data; name="avatar"; filename="../exploit.php"
```

The response reports that `avatars/exploit.php` was uploaded — the plain `../` was stripped out before saving. The defense saw the literal sequence and removed it.

<img width="2704" height="1394" alt="step-9" src="https://github.com/user-attachments/assets/feb7a945-719b-4c76-9cf4-5dbab204048c" />

### Step 3: Encode the Slash

Obfuscate the traversal by URL-encoding only the slash:

```http
Content-Disposition: form-data; name="avatar"; filename="..%2fexploit.php"
```

Now the confirmation message reads `avatars/../exploit.php` — proof the server **URL-decoded the filename after** its traversal check, restoring the separator the check never saw. The file has been written one directory up, into `/files/`, where the no-execution rule does not apply.

<img width="2903" height="1168" alt="step-10" src="https://github.com/user-attachments/assets/945ec4e7-3a5b-406c-bb66-fc48eb935608" />

### Step 4: Execute the Escaped Shell

Request the shell at its new location:

```http
GET /files/exploit.php
```

This time the server executes it, and the response body is the contents of `/home/carlos/secret`. Submit the secret via the lab banner. The lab is solved.

<img width="1969" height="270" alt="step-11" src="https://github.com/user-attachments/assets/d6b17afb-3c5c-46c6-a943-7acabf529d4c" />

<img width="3031" height="747" alt="step-12" src="https://github.com/user-attachments/assets/f56a5a12-4582-4f1c-b679-99ba1faa68b1" />

<img width="3150" height="1101" alt="step-13" src="https://github.com/user-attachments/assets/73a885c6-32e3-428a-bdda-c91ab92f9711" />

## Key Takeaways

- "Stored safely" is a property of the final path, not of the upload form. Anything in the filename that can alter the destination — separators, encodings, absolute paths — is part of the security boundary.
- Check the order of operations: validation ran on the *encoded* string, the filesystem received the *decoded* one. Decode-then-validate is safe; validate-then-decode is a bypass waiting to happen.
- The server's own confirmation message was the oracle: comparing `avatars/exploit.php` with `avatars/../exploit.php` told us precisely when the traversal survived.

## Remediation

Never use the client-supplied filename to build a storage path: save uploads under server-generated random names, and resolve the final absolute path to confirm it stays inside the intended directory (canonicalization check) before writing. Decode all encodings *before* validating, and apply the same execution-disabled hardening to every directory an upload could conceivably reach.
