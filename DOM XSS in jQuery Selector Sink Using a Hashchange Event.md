# DOM XSS in jQuery Selector Sink Using a Hashchange Event

**Platform:** PortSwigger Web Security Academy
**Category:** Cross-Site Scripting (XSS)
**Difficulty:** Apprentice
**Status:** ✅ Solved

## Background

The blog homepage has a convenience feature: when the URL fragment changes, JavaScript reads `location.hash` and uses jQuery's `$()` to find the matching post element and scroll to it. The flaw is in what `$()` does with its argument. Given a selector like `#post1`, it looks an element up. But given a string that *looks like HTML* — starting with `<` — jQuery (in affected versions) treats it as markup and **creates the elements**. The page passes the hash, which is fully attacker-controlled, straight into `$()` inside a `hashchange` event handler. So a fragment of `#<img src=x onerror=print()>` is not searched for at all: it is built as an image element, its `src` fails to load, and the `onerror` handler executes in the page's origin. Because the trigger is a `hashchange`, the exploit can be fired *after* the page has loaded, without any reload — which is exactly what the delivery iframe below arranges.

## Lab Objective

The blog homepage is vulnerable to DOM XSS through the `location.hash` source and the jQuery selector sink. Use the exploit server to deliver an iframe that changes the hash in the victim's browser and calls the `print()` function, solving the lab.

**Provided resources:** the lab blog application and an exploit server.

<img width="2155" height="1033" alt="step-1" src="https://github.com/user-attachments/assets/145377cd-6081-4e33-a6ff-2f022f179191" />


## Walkthrough

### Step 1: Understand the Target Behaviour

The homepage's script listens for `hashchange` events and feeds the new fragment into jQuery's selector to locate a blog post. Any string after `#` reaches `$()` unchecked — including strings that begin with `<`, which jQuery will parse as HTML instead of a selector. The payload therefore has to arrive as a *hash change on an already-loaded page*, not as part of the initial navigation alone.

<img width="3052" height="1511" alt="step-2" src="https://github.com/user-attachments/assets/0c92ba74-15d9-4496-b8b9-4d7fc1653b7d" />


### Step 2: Build the Delivery iframe on the Exploit Server

Open the exploit server and place this in the **Body** section, with your lab URL in place of the placeholder:

```html
<iframe src="https://YOUR-LAB-ID.web-security-academy.net/#" onload="this.src+='<img src=x onerror=print()>'"></iframe>
```

The iframe does three jobs in sequence:

1. `src="…/#"` loads the vulnerable homepage inside the frame, with an empty fragment.
2. `onload` fires once that page has loaded — the page's `hashchange` listener is now armed.
3. `this.src += '<img src=x onerror=print()>'` appends the payload to the frame's URL as its new fragment. Changing only the fragment does not reload the page, but it *does* fire `hashchange` inside it.

The framed page then reads the new hash and hands `<img src=x onerror=print()>` to `$()`. jQuery builds the image element, the bogus `src=x` fails, and `onerror` calls `print()`.

<img width="3095" height="1008" alt="step-3" src="https://github.com/user-attachments/assets/e54a1ac0-b0dd-4fab-aae7-830b3724bdd5" />


### Step 3: Store and Test the Exploit

Click **Store**, then **View exploit** to open your exploit page yourself. The print dialog appears, confirming the full chain works end to end: iframe load → hash appended → `hashchange` → jQuery parses the fragment as HTML → `onerror` executes.

<img width="3350" height="1856" alt="step-4" src="https://github.com/user-attachments/assets/acfd8af4-0e92-4e74-99b8-69b5ac09fb33" />


<img width="3137" height="1557" alt="step-5" src="https://github.com/user-attachments/assets/b384527b-e529-47ca-abe2-a2aa1e494ff7" />


### Step 4: Cross-Check the Mechanism Directly

The same sink can be verified without the iframe by visiting the lab homepage with the payload as the fragment:

```text
https://YOUR-LAB-ID.web-security-academy.net/#<img src=x onerror=print()>
```

and then changing the hash so the `hashchange` handler runs. The browser's print dialog opens over the blog homepage — direct evidence that the fragment, the handler and the selector sink are the whole vulnerability; the iframe is only the delivery vehicle.

<img width="3056" height="1798" alt="step-6" src="https://github.com/user-attachments/assets/3b8628a8-2209-4e94-8ef8-d1f55bee9294" />


### Step 5: Deliver to the Victim

Back on the exploit server, click **Deliver exploit to victim**. The simulated victim loads the exploit page, the iframe fires the payload in their browser, `print()` is called, and the lab is solved.

<img width="2121" height="1317" alt="step-7" src="https://github.com/user-attachments/assets/50ff98d6-d25b-4cdd-97c5-c0b24e4c3b54" />


<img width="2193" height="1003" alt="step-8" src="https://github.com/user-attachments/assets/859f2d8f-7cca-475e-ba13-d7a0297b91fd" />


## Key Takeaways

- jQuery's `$()` is two functions in one: selector lookup *and* HTML construction. Feeding it untrusted input lets the input choose which — a string starting with `<` becomes DOM, with event handlers live.
- The source here (`location.hash`) never reaches the server, so no server-side filter or log will ever see the payload; DOM vulnerabilities must be found by reading the client-side code paths from source to sink.
- The `hashchange` event is the trigger that makes post-load delivery possible: changing a fragment fires an event without a reload, so an iframe's `onload` handler can arm the page first and detonate the payload second.
- The exploit's shape follows the sink: no quotes to break out of and no tags to close — the payload *is* a complete HTML element, because the sink parses HTML.

## Remediation

Never pass untrusted strings to jQuery's `$()` or equivalent DOM-construction helpers. If a hash is meant to identify an element, validate it against a strict pattern (for example, an allowlist of known post IDs, or a regex for a safe identifier) and use `document.getElementById` / `querySelector` with the validated value, escaping it as a selector rather than letting it be parsed as markup. Keep jQuery updated — behaviour around HTML-looking strings has been tightened across versions — and audit all `hashchange`/`location` consumers for sinks (`$()`, `innerHTML`, `document.write`) as a class. A Content Security Policy that restricts inline event handlers can reduce impact, but validating the fragment before it reaches the sink is the actual fix.
