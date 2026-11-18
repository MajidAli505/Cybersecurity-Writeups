# Stored XSS into Anchor href Attribute with Double Quotes HTML-Encoded

**Platform:** PortSwigger Web Security Academy
**Category:** Cross-Site Scripting (XSS)
**Difficulty:** Apprentice
**Status:** ✅ Solved

## Background

Blog comments are stored and re-displayed to every visitor — the definition of stored XSS surface. In this blog, a commenter's **Website** field is rendered as the `href` of the link on their name:

```html
<a href="WEBSITE_VALUE">Author Name</a>
```

The application HTML-encodes double quotes, so the obvious move — breaking out of the attribute with a `"` and adding an event handler — is closed. But breaking out was never the only option, because the *value itself* is dangerous. An `href` does not have to point at a web page: the `javascript:` scheme makes the link execute script when clicked. If the application stores the Website value without validating its scheme, the attacker doesn't need to escape the attribute at all. They simply supply, as the entire value, a URL that *is* code:

```html
<a href="javascript:alert(1)">xyz</a>
```

The trap is stored in the database with the comment and waits, indefinitely, for any reader who clicks the author's name.

## Lab Objective

The blog's comment feature stores the Website field and renders it into an anchor `href` attribute, with double quotes HTML-encoded. Store a comment whose author link executes `alert(1)` when clicked, and trigger it to solve the lab.

**Provided resources:** the lab blog application and its comment function.

<img width="2175" height="866" alt="step-1" src="https://github.com/user-attachments/assets/65e4c904-a39f-4df9-a702-e4d8dbd4a56c" />


## Walkthrough

### Step 1: Open a Post and Study the Comment Form

Open a blog post and find the **Leave a comment** form. Alongside the comment text it asks for **Name**, **Email** and **Website** — three stored fields, each a candidate for a different rendering context.

<img width="3141" height="1661" alt="step-2" src="https://github.com/user-attachments/assets/59d1e006-5102-451b-a928-2e37fbf8f86d" />


<img width="2594" height="1333" alt="step-3" src="https://github.com/user-attachments/assets/f8003da2-f5ee-4fd3-bfa9-3589afa522b6" />


### Step 2: Post a Test Comment and Find the Rendering Context

Submit an ordinary comment first (name `xyz`, a normal website) and view the post. The commenter's name is rendered as a **link**, and inspecting it shows the Website value sitting inside the anchor's `href` attribute. That is the context: not the comment body, not the name text — the *destination of the author link*.

The encoding check matters too: double quotes in the input are HTML-encoded, so the attribute cannot be broken out of. The attack has to work from inside the value.
<img width="2284" height="1591" alt="step-4" src="https://github.com/user-attachments/assets/1c54d74e-9d26-4629-a88d-dd203295a2b8" />


### Step 3: Store a `javascript:` URL as the Website

Post a second comment, this time setting the **Website** field to a JavaScript URL:

```text
javascript:alert(1)
```

The name and comment text can be anything. Submit the comment — it is stored, and the author name on the post is now a link whose `href` is the script URL:

```html
<a href="javascript:alert(1)">xyz</a>
```

No quote was needed, no tag was created, and the double-quote encoding is irrelevant: the payload never leaves the attribute value, because the value itself is the payload.

<img width="2105" height="1579" alt="step-5" src="https://github.com/user-attachments/assets/ca3ceeca-4091-47b9-a1b4-33412225e6f7" />


### Step 4: Click the Author Link

Return to the blog post and click the comment author's name (**xyz**). The browser follows the `javascript:` URL and executes `alert(1)` in the blog's origin. The stored payload has fired — for this visitor, and it would fire identically for every future reader who clicks. The lab is solved.

<img width="2955" height="1274" alt="step-6" src="https://github.com/user-attachments/assets/618259c7-4252-4541-8e15-01f887e3b0fb" />


<img width="3050" height="1099" alt="step-7" src="https://github.com/user-attachments/assets/a5b966e1-c7c3-472e-9b16-c244d5fc2e1f" />


## Key Takeaways

- Stored XSS persists: the payload is saved once and served to every viewer of the page, which is what makes stored variants generally more severe than reflected ones.
- Encoding quotes prevents *breaking out of* an attribute; it does nothing about a value that is malicious *as a whole*. URL attributes are only safe if the scheme is validated — `javascript:` (and `data:`) must be rejected, not encoded.
- The recon step decided the payload: posting a benign comment and inspecting the rendered HTML revealed that the Website field lands in an `href`, and that one fact — not a payload list — produced the exploit.
- This sink needs a click, so it is dormant rather than automatic; in a real blog that is still a reliable trigger, because the author-name link is exactly what curious readers click.

## Remediation

Validate URL fields by scheme: parse the value as a URL and allow only `http:` and `https:` (or relative URLs), rejecting `javascript:`, `data:` and anything else before storage. Apply context-aware output encoding as a second layer — quotes encoded in attribute contexts — while remembering it does not substitute for scheme validation. Prefer rendering commenter websites as plain text, or through a redirect/warning interstitial, when a link is not essential. Stored content deserves the same scrutiny on *output* as on input, since it is served to other users long after submission.
