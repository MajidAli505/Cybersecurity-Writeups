# Reflected XSS into a JavaScript String with Angle Brackets HTML Encoded

**Platform:** PortSwigger Web Security Academy
**Category:** Cross-Site Scripting (XSS)
**Difficulty:** Apprentice
**Status:** ✅ Solved

## Background

The search results page contains an analytics snippet that embeds the search term directly into an inline script:

```javascript
var searchTerms = 'YOUR_SEARCH';
document.write('<img src="/resources/images/tracker.gif?searchTerms=' + encodeURIComponent(searchTerms) + '">');
```

The input therefore lands in a **JavaScript string context**, inside a single-quoted literal. The application HTML-encodes angle brackets, which is a defence for the wrong context: no new HTML element is needed here at all, because the input is already inside code. What matters is the quote. A `'` in the search term terminates the string literal early, and whatever follows is parsed as JavaScript in its own right. The elegant part is the payload shape: instead of breaking out and starting a new statement, the payload turns the assignment itself into an arithmetic expression —

```javascript
var searchTerms = ''-alert(1)-'';
```

— an empty string, minus the return value of `alert(1)`, minus another empty string. JavaScript evaluates the whole expression to produce the variable's value, and evaluating it means calling `alert(1)`.

## Lab Objective

The blog search term is reflected into a JavaScript string in an inline script, with angle brackets HTML-encoded. Escape the string context and execute `alert(1)` to solve the lab.

**Provided resources:** the lab blog application and its search function.

<img width="2192" height="1026" alt="step-1" src="https://github.com/user-attachments/assets/3a90e3ba-1562-4bbe-bf74-02e44f53d7cc" />


## Walkthrough

### Step 1: Search a Canary and Find Every Reflection

Search for a unique test string (`test123`). It appears in the results heading — and, more importantly, inside the page. A search term on a results page is often reflected in more than one place; the heading is the visible one, not necessarily the exploitable one.

<img width="3203" height="1602" alt="step-2" src="https://github.com/user-attachments/assets/36d14759-5d9f-4bd6-947e-43c63ea08224" />


### Step 2: Inspect the Page Source — the Script Reflection

Open the page in Developer Tools and search the DOM for the canary. Alongside the heading, it sits inside an inline `<script>` block:

```javascript
var searchTerms = 'test123';
document.write('<img src="/resources/images/tracker.gif?searchTerms=' + encodeURIComponent(searchTerms) + '">');
```

The context is now precise: single-quoted JavaScript string, in a script that runs on page load. Angle brackets are encoded, but they are irrelevant — the attack never needs to create markup. It needs to end the string.

<img width="2629" height="1088" alt="step-3" src="https://github.com/user-attachments/assets/230d1454-c6ba-4c12-9b86-9dc84cba61fe" />


### Step 3: Break Out of the String with an Expression Payload

Submit the following as the search term:

```text
'-alert(1)-'
```

Read what the script becomes after reflection:

```javascript
var searchTerms = ''-alert(1)-'';
```

- The payload's first `'` closes the string immediately, leaving an empty string `''`.
- `-alert(1)-` is now JavaScript code positioned between two string literals: the minus operators force the expression to be evaluated, and evaluating it calls `alert(1)`.
- The payload's final `'` opens a new string that the application's own closing quote terminates, keeping the syntax valid.

No semicolon, no new statement, no angle brackets — the existing statement is simply bent into executing the call while it computes a (nonsense, `NaN`) value for `searchTerms`.

<img width="2469" height="783" alt="step-4" src="https://github.com/user-attachments/assets/86703727-92a2-46eb-b639-ae8b21954431" />


### Step 4: The Alert Fires

Reloading with the payload — the URL shows `?search=%27-alert%281%29-%27` — the inline script runs during page load, the expression is evaluated, and `alert(1)` fires. The lab is solved.
<img width="2942" height="811" alt="step-5" src="https://github.com/user-attachments/assets/ade7bf2e-f38a-4af4-bbcf-42bd6353124b" />


## Key Takeaways

- A reflection inside a `<script>` block is a code context, not a markup context: HTML-encoding angle brackets defends against a threat that does not exist there, while the quote that actually delimits the context is left open.
- The payload did not inject a new statement; it reshaped the existing one. `''-alert(1)-''` works because JavaScript evaluates function calls inside expressions — a useful pattern whenever the breakout point sits mid-expression and statement separators are unavailable or filtered.
- Finding the *second* reflection was the whole game: the visible heading reflection was a dead end, and only inspecting the page source revealed the script context. Always enumerate every place a canary lands.
- `document.write` with a tracker image is a realistic pattern — analytics and tracking snippets are a common place for search terms to be embedded into scripts verbatim.

## Remediation

Never embed unescaped user input into inline scripts. If a value must reach JavaScript, pass it through a proper JSON/JavaScript encoder that escapes quotes, backslashes and line terminators (`'` → `\u0027`, and so on), or deliver it via a data attribute / JSON block parsed with `JSON.parse` rather than as raw code. HTML-encoding is not a substitute: each context — HTML, attribute, JavaScript, URL — has its own encoding rules, and applying the HTML set inside a script block leaves the context's real delimiters untouched. Moving the tracking call to an external script that reads the term from the URL with `URLSearchParams` and treats it strictly as data removes the code-context reflection entirely.
