# Web Shell Upload via Obfuscated File Extension

**Platform:** PortSwigger Web Security Academy
**Category:** File Upload
**Difficulty:** Practitioner
**Status:** ✅ Solved

## Background

An extension check is only as strong as the agreement between the components that read the filename. The application that validates an upload, the framework that stores it, and the filesystem and web server that later open it do not necessarily parse a filename the same way. A URL-encoded null byte exploits exactly that disagreement. In a name such as `shell.php%00.jpg`, one layer decodes the `%00` and sees a string that ends in `.jpg`, while a lower layer treats the null byte as the end of the string and stores the file as `shell.php`. The check and the storage are both "working" — they are simply working on two different filenames. That mismatch, not the payload, is the vulnerability in this lab.

## Lab Objective

The site's image upload function blacklists certain file extensions, but the blacklist can be bypassed with a classic obfuscation technique. Upload a basic PHP web shell, use it to read `/home/carlos/secret`, and submit the secret via the lab banner.

**Provided resources:** your account `wiener:peter`.

<img width="2190" height="1099" alt="step-1" src="https://github.com/user-attachments/assets/694cc347-717a-466d-baf4-b49349de95e0" />


<img width="3305" height="1588" alt="step-2" src="https://github.com/user-attachments/assets/fdc0105d-8764-49a0-a565-3fcd83ed3d14" />


## Walkthrough

### Step 1: Find the Upload and Try a Plain PHP Shell

Log in as `wiener` and open your account page. It offers an avatar upload — the vulnerable function.

<img width="2664" height="1389" alt="step-3" src="https://github.com/user-attachments/assets/a4b0b30e-c721-4e6e-b067-de8ed35e50d3" />


Create the standard one-line shell and select it for upload, under the name `pann.php`:

```php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

<img width="2207" height="849" alt="step-4" src="https://github.com/user-attachments/assets/b48a48d8-ffbd-4dac-9efe-1050fbcdcfab" />


The upload is rejected outright:

> Sorry, only JPG & PNG files are allowed

<img width="1777" height="444" alt="step-5" src="https://github.com/user-attachments/assets/350b4407-2021-4b34-99f2-69a41a2697f1" />


The same attempt in Burp confirms what happened. The request carries `filename="pann.php"` with the PHP payload as the file contents, and the server answers **403 Forbidden** with the same "only JPG & PNG files are allowed" message. A plain `.php` extension has no way through this check.

<img width="2741" height="1074" alt="step-6" src="https://github.com/user-attachments/assets/678a22b3-d550-4923-a875-95a6d6f7a36d" />


### Step 2: Obfuscate the Extension with a URL-Encoded Null Byte

Several obfuscation techniques were tried against this check without success. The one that worked was the URL-encoded null byte. Send the upload request to Burp Repeater and change only the filename:

```text
filename="pann.php%00.jpg"
```

The file contents stay exactly the same PHP shell. The idea is the split described above: after URL decoding, the validation sees a name ending in the harmless `.jpg`, but the null byte (`%00`) terminates the name for the layer that writes the file to disk, so it is stored under everything before the null byte — `pann.php`.

This time the response is **200 OK**, and it gives the trick away in plain language:

> The file avatars/pann.php has been uploaded.

The server itself confirms the file was saved as `pann.php`, not as a `.jpg`. The obfuscated extension passed the check, and the stored file is a PHP script.

<img width="2713" height="991" alt="step-7" src="https://github.com/user-attachments/assets/0550c1ab-fe33-4547-8052-6b02411024f3" />


### Step 3: Execute the Shell

Uploaded avatars are served from a predictable location, so request the stored file directly:

```http
GET /files/avatars/pann.php
```

Because the file on disk really ends in `.php`, the server executes it instead of serving it as an image. The response body is the contents of `/home/carlos/secret`, printed by the shell.

<img width="2577" height="994" alt="step-8" src="https://github.com/user-attachments/assets/6c2817fb-0cb7-45c6-8a08-47eb3dde283b" />


### Step 4: Submit the Secret

Copy the secret from the response and submit it using the button in the lab banner. The lab is solved.

<img width="3493" height="884" alt="step-9" src="https://github.com/user-attachments/assets/782a0b59-ac0c-4dab-b4b3-139e122d9e6f" />


<img width="3026" height="790" alt="step-10" src="https://github.com/user-attachments/assets/18d1ae54-66ac-42f0-bf84-9d7d8da09f2d" />


## Key Takeaways

- Validation and storage must operate on exactly the same filename. Any decoding or truncation that happens *after* the check — URL decoding, null-byte termination, path normalisation — reopens every extension the check just blocked.
- The server's own success message was the confirmation: it reported the upload as `pann.php`. Response messages that echo the stored path or filename are useful reconnaissance for an attacker.
- A null byte is only the best-known member of a family. Double extensions, trailing dots and spaces, case variation, and alternate encodings all test the same question: does every layer agree on what this file is called?

## Remediation

Canonicalise and fully decode the filename *before* any validation, and reject names containing null bytes or other control characters outright. Validate the extension on the final, stored name — ideally by discarding the user's filename entirely and generating a random server-side name with an allowlisted extension. Combine this with server-side content validation, store uploads outside the web root or with script execution disabled, and never let the stored location execute files based on a user-supplied name.
