# Reflected XSS into Attribute with Angle Brackets HTML-Encoded

**Platform:** PortSwigger Web Security Academy
**Category:** Cross-Site Scripting (XSS)
**Difficulty:** Apprentice
**Status:** ✅ Solved

## Background

This lab is the standard lesson that **encoding is context-specific**. The blog's search term is reflected inside the `value` attribute of an `<input>` element:

```html
<input type="text" value="YOUR_SEARCH">
```

The application HTML-encodes `<` and `>` in the input, so a classic payload like `<script>alert(1)</script>` is neutralised — it arrives as `&lt;script&gt;…` and can never become a tag. But the encoding stops there: the **double quote is not encoded**. Since the input sits inside a double-quoted attribute, a `"` in the search term closes the `value` attribute early, and everything after it is parsed as *more attributes of the same tag*. Attributes include event handlers — and two of them combine into a self-triggering payload: `autofocus`, which makes the browser focus the element as soon as the page loads, and `onfocus`, which runs JavaScript the moment that focus lands. No new tag is ever created, so the angle-bracket encoding never gets a chance to matter.

## Lab Objective

The blog search reflects input into an HTML attribute with angle brackets HTML-encoded. Break out of the attribute and execute `alert(1)` to solve the lab.

**Provided resources:** the lab blog application and its search function.

<img width="2160" height="840" alt="step-1" src="https://github.com/user-attachments/assets/49be836e-57da-4b88-94c9-77c6a0ec46ac" />


## Walkthrough

### Step 1: Identify the Reflection Context

Search the blog for a test string and inspect how it is reflected. The term appears inside the search field's markup as the value of an attribute:

```html
<input type="text" value="YOUR_SEARCH">
```

Two facts define the attack:

1. `<` and `>` are HTML-encoded — injecting a new element is impossible.
2. The value is wrapped in double quotes — so the interesting question is whether the quote character itself survives. It does.

### Step 2: Break Out of the Attribute and Add an Event Handler

Submit this as the search term:

```text
" autofocus onfocus=alert(1) x="
```

Piece by piece:

- The leading `"` **closes** the `value` attribute, which is now empty.
- `autofocus` is parsed as a new attribute of the `<input>` — the browser will focus this field automatically on page load.
- `onfocus=alert(1)` adds a second new attribute: an event handler that runs `alert(1)` when the element receives focus.
- The trailing `x="` opens a dummy attribute that absorbs the original closing quote, keeping the tag's syntax valid.

The reflected markup becomes:

```html
<input value="" autofocus onfocus=alert(1) x="">
```

No angle brackets were used anywhere — the payload lives entirely inside an existing tag, which is why the encoding defence never engages.

### Step 3: The Payload Fires Itself

When the results page loads, the browser applies `autofocus` to the search field; the focus event fires immediately; `onfocus` executes `alert(1)`. The alert appears over the results page — which also shows the full payload reflected in the "0 search results for …" heading — and the lab is solved.

<img width="3059" height="1390" alt="step-2" src="https://github.com/user-attachments/assets/f89610a0-a929-44db-b001-5eb02d69fc95" />


<img width="3048" height="1044" alt="step-3" src="https://github.com/user-attachments/assets/ebe94428-6611-4a17-a374-dadad2541277" />


## Key Takeaways

- HTML-encoding `<` and `>` only defends the *HTML content* context. Inside an attribute, the dangerous character is the quote that delimits the attribute — if that is not encoded, the attribute can be escaped.
- Once you can add attributes, event handlers (`onfocus`, `onclick`, …) are code execution, and pairing a handler with `autofocus` removes the need for any user interaction.
- The trailing `x="` is not decoration: it consumes the application's own closing quote so the resulting tag stays well-formed and the injected attributes are parsed as intended.
- Context decides the payload. The same search box with the same encoding would need a completely different approach if the reflection were in element content or in a JavaScript string — which is why the first step is always *find the exact context*, not *try payloads*.

## Remediation

Encode for the context the data lands in. For attribute contexts, that means encoding quotes (both `"` and `'`) in addition to angle brackets — or better, using a templating framework that applies context-aware output encoding automatically. Avoid reflecting user input into attributes at all where possible, and never into event-handler or URL attributes. As defence in depth, a Content Security Policy can block inline event handlers, but correct contextual encoding at the reflection point is the fix that removes the vulnerability.
