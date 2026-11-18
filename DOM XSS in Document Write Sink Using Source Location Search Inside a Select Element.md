# DOM XSS in Document Write Sink Using Source Location Search Inside a Select Element

**Platform:** PortSwigger Web Security Academy
**Category:** Cross-Site Scripting (XSS)
**Difficulty:** Apprentice
**Status:** ✅ Solved

## Background

The product page builds its store selector in JavaScript. It reads the `storeId` parameter from the query string and writes it into the page with `document.write`, wrapped as the pre-selected option of a `<select>` element:

```javascript
var stores = ["London", "Paris", "Milan"];
var store = (new URLSearchParams(window.location.search)).get('storeId');
document.write('<select name="storeId">');
if (store) {
    document.write('<option selected>' + store + '</option>');
}
for (var i = 0; i < stores.length; i++) {
    if (stores[i] === store) { continue; }
    document.write('<option>' + stores[i] + '</option>');
}
document.write('</select>');
```

Source (`location.search`) flows into sink (`document.write`) with no sanitization — textbook DOM XSS. The one complication is *where* the value lands: inside an `<option>`, inside a `<select>`. A `<select>` element's content model is restrictive — the HTML parser only permits options, option groups and a few script-supporting elements inside it, and discards or relocates other markup such as an `<img>` tag. Injecting an element directly therefore fails. The payload must first **close the option and the select**, stepping out of the restricted context, and only then introduce the executing element.

## Lab Objective

The product page's stock checker is vulnerable to DOM XSS: the `storeId` parameter is written into a `<select>` element via `document.write`. Break out of the select element and call `alert(1)` to solve the lab.

**Provided resources:** the lab shop application and its product pages.

<img width="2203" height="1068" alt="step-1" src="https://github.com/user-attachments/assets/772d0863-8d2e-4838-8c45-d52668e05dc5" />

## Walkthrough

### Step 1: Find the Feature — the Store Selector

Open a product page. Above the **Check stock** button sits a dropdown of stores (London, Paris, Milan). Choosing a store reloads the page with that store in the query string:

```text
/product?productId=1&storeId=London
```

That parameter is the input to test.

<img width="3044" height="1755" alt="step-2" src="https://github.com/user-attachments/assets/e614632f-6f33-484f-adae-d6e20435abcc" />

<img width="2416" height="932" alt="step-3" src="https://github.com/user-attachments/assets/95177682-db96-4407-8e9f-50a9a42e02b3" />

### Step 2: Read the Client-Side Code

Inspect the page in Developer Tools and find the inline script that builds the selector (shown in full in the Background above). It confirms everything an attacker needs:

- **Source:** `storeId` is read from `window.location.search` via `URLSearchParams`.
- **Sink:** the raw value is concatenated into `document.write('<option selected>' + store + '</option>')` — written into the DOM as HTML, unescaped.
- **Context:** the value lands inside `<option>…</option>`, inside `<select>…</select>`.

<img width="2465" height="1167" alt="step-4" src="https://github.com/user-attachments/assets/7dc752e9-fb4e-42e9-aad6-8ee4b4f6b6ef" />


### Step 3: Break Out of the Select, Then Inject

A payload placed inside the `<select>` cannot survive as an element — the parser will not honour an `<img>` between options. So the payload first closes the two open elements, then adds the executing element:

```text
/product?productId=1&storeId=</option></select><img src=x onerror=alert(1)>
```

After the script writes it, the document stream reads:

```html
<select name="storeId">
    <option selected></option></select><img src=x onerror=alert(1)></option>
    …
```

The injected `</option></select>` ends the restricted context early. The `<img>` that follows is now ordinary body markup: it is parsed as a real element, its `src=x` fails to load, and the `onerror` handler executes `alert(1)` — all during page load, because `document.write` inserts the content into the parser stream exactly where the script sits.

### Step 4: The Alert Fires

Loading the crafted URL produces the alert, and the lab is solved. The leftover closing tags the page writes afterwards are harmless — the parser tolerates the stray markup, and the payload has already run.

## Key Takeaways

- `document.write` is one of the classic DOM XSS sinks because it feeds its argument to the HTML parser as markup. Any source reaching it — here `location.search` — must be treated as code, not text.
- Context still rules inside a sink: the same payload that works in a `<div>` fails inside a `<select>`, whose content model rejects most elements. Closing the restrictive parent elements (`</option></select>`) is the standard escape.
- The vulnerable code was fully visible in the page: reading the inline script gave the source, the sink, the exact surrounding markup, and therefore the exact breakout — no fuzzing required.
- This pattern (pre-selecting a value from the URL by writing it into a form control) is a realistic feature; the bug is not the feature but writing unvalidated input as HTML instead of setting the control's value as data.

## Remediation

Do not use `document.write` with user-influenced data — build the control with DOM APIs instead: create the `<select>` and its `<option>` elements in markup or with `document.createElement`, and set the pre-selected option by comparing values and assigning the `selected` property, so the parameter is only ever handled as a string value. Validate `storeId` against the allowlist of real store names (here, the `stores` array already exists — any value not in it should be ignored), which alone would have neutralised the payload. As a general rule, audit every `location.*` consumer for HTML sinks (`document.write`, `innerHTML`, jQuery `html()`/`$()`), and prefer `textContent` whenever the intent is to display text.
