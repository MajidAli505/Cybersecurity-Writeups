# DOM XSS in jQuery Anchor href Attribute Sink Using Location Search Source

**Platform:** PortSwigger Web Security Academy
**Category:** Cross-Site Scripting (XSS)
**Difficulty:** Apprentice
**Status:** ✅ Solved

## Background

DOM-based XSS lives entirely in the browser: a **source** the attacker controls (here, `location.search` — the page's query string) is read by client-side JavaScript and written into a **sink** that can execute code. This lab's sink is subtle. The page uses jQuery to copy the `returnPath` query parameter into the `href` attribute of a "Back" link:

```javascript
$('#backLink').attr("href", (new URLSearchParams(window.location.search)).get('returnPath'));
```

Nothing executes when the page loads — an `href` is just an address. But an anchor's `href` accepts more than web addresses: the `javascript:` scheme turns the link itself into code. When a user clicks a link whose `href` is `javascript:alert(document.cookie)`, the browser executes that script in the page's context. So whoever controls `returnPath` controls what the Back link *does* — and a victim only has to be persuaded to visit the crafted URL and click.

## Lab Objective

The Submit feedback page is vulnerable to DOM XSS via the `returnPath` parameter, which is written to the back link's `href`. Craft a URL that makes the link execute `alert(document.cookie)` when clicked, and trigger it to solve the lab.

**Provided resources:** the lab blog application with its Submit feedback page.

<img width="2218" height="753" alt="step-1" src="https://github.com/user-attachments/assets/1354b530-e256-41b9-b09f-a5741a1bcaee" />

## Walkthrough

### Step 1: Open the Submit Feedback Page

Go to the **Submit feedback** page and look at its URL — it carries a query string, and the page contains a **Back** link. Both are about to matter.

<img width="3112" height="1704" alt="step-2" src="https://github.com/user-attachments/assets/87dcf019-dc8e-4815-b749-15e581b0f40f" />

### Step 2: Plant a Canary in `returnPath`

Change the `returnPath` query parameter to `/` followed by a random alphanumeric string (for example `/rtgt`) and load the page. A harmless, unique value first: before exploiting anything, prove where the input lands.

<img width="2389" height="1207" alt="step-3" src="https://github.com/user-attachments/assets/374056c1-21ee-4fcf-934b-1791ebc3b5e4" />

### Step 3: Inspect the Back Link — and Read the Page's Own Code

Right-click the **Back** link and inspect the element. The canary is sitting inside the anchor's `href` attribute:

```html
<a id="backLink" href="/rtgt">Back</a>
```

The inline script right below it shows the exact mechanism — jQuery selects `#backLink` and sets its `href` attribute from `window.location.search`'s `returnPath` parameter, with no validation of the value. Source and sink are both confirmed in one screenshot: the URL fully controls the link's destination.

<img width="2944" height="473" alt="step-4" src="https://github.com/user-attachments/assets/7dee35ce-9545-4a0a-87c6-6907b859f0df" />

### Step 4: Weaponize the Link with a `javascript:` URL

Set `returnPath` to a script URL instead of a path:

```text
/feedback?returnPath=javascript:alert(document.cookie)
```

Press Enter to load the page with this parameter. The page looks unchanged — but the Back link's `href` is now executable code waiting for a click.

<img width="2732" height="1473" alt="step-5" src="https://github.com/user-attachments/assets/bfa3f60b-60c1-430b-922b-55ab5296274a" />

<img width="2790" height="1262" alt="step-6" src="https://github.com/user-attachments/assets/2ab0cce4-ef0c-4173-a9c8-69961938217c" />

### Step 5: Click Back and Trigger the Alert

Click the **Back** link. The browser treats the `javascript:` URL as a script and runs it in the page's origin: an alert fires displaying `document.cookie`. The lab registers the execution and is solved.

<img width="3121" height="779" alt="step-7" src="https://github.com/user-attachments/assets/6da65cc4-043a-4131-99ec-2304154d41e9" />

<img width="2088" height="1352" alt="step-8" src="https://github.com/user-attachments/assets/1385693b-90e5-4914-856e-c51d80759978" />

## Key Takeaways

- DOM XSS analysis is source-to-sink tracing: the canary proved `returnPath` reached the `href`, and the page's own script named both the source (`location.search`) and the sink (jQuery's `attr("href", …)`).
- Not every sink fires on page load. An `href` sink is *dormant* until clicked — the payload is the link's destination, and the trigger is user interaction, which is why this attack is delivered as a link a victim is lured into opening and clicking.
- The `javascript:` scheme is what converts an attribute value into code execution. Any sink that sets a navigable attribute (`href`, `src` of certain elements, form actions) must treat scheme control as code control.
- The canary-first method (`/rtgt`) separated the two questions — *does my input reach a sink?* and *what payload does this sink need?* — instead of firing exploit strings blindly.

## Remediation

Never copy unvalidated URL parameters into navigable attributes. If a return path is needed, validate it against an allowlist of expected paths or require it to be a relative URL beginning with a single `/`, and explicitly reject values with a scheme — above all `javascript:` and `data:`. Where a library sets attributes, prefer setting properties with validated values and consider a sanitizer for any URL handling. Defence in depth: a restrictive Content Security Policy does not stop `javascript:` links clicked by users in all cases, so validation at the sink is the real fix; HttpOnly cookies additionally limit what a successful script can read, though they do not prevent the execution itself.
