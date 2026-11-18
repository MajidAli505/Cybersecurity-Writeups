# Detecting NoSQL Injection

**Platform:** PortSwigger Web Security Academy
**Category:** NoSQL Injection
**Difficulty:** Apprentice
**Status:** ✅ Solved

## Background

NoSQL databases such as MongoDB use query syntax built from structured data (JSON/BSON-like objects) rather than a string query language — but that does not make them immune to injection. One classic MongoDB pattern evaluates a JavaScript expression for each document, via the `$where` operator or similar server-side scripting. If user input is concatenated into that JavaScript string without sanitization, the input stops being data and becomes code: quotes break out of the string literal, and operators such as `&&` and `||` let the attacker rewrite the condition the database evaluates. This lab is about *detection*: proving, step by step, that a parameter is evaluated as JavaScript — first with a syntax error, then with controlled boolean conditions, and finally with an always-true condition that changes what data comes back.

## Lab Objective

The shop's product filter looks up products using a NoSQL database query that is vulnerable to NoSQL injection. Detect the injection in the `category` filter parameter and exploit it to view unreleased products, which are normally hidden by the filter.

**Provided resources:** the lab shop application only — no account is needed.

<img width="2191" height="930" alt="step-1" src="https://github.com/user-attachments/assets/87c92c9b-5bc7-457f-b544-213c4e71e66e" />


## Walkthrough

### Step 1: Find the Filter Request

On the shop homepage, select a category filter (here, **Gifts**). The page loads products through a request like:

```http
GET /filter?category=Gifts
```

Send this request to Burp Repeater. The `category` parameter is the test target: it is the value most likely to be placed inside the database query.

<img width="3094" height="1785" alt="step-2" src="https://github.com/user-attachments/assets/a344d7c3-b138-431b-8489-3f996be16663" />


<img width="3401" height="1832" alt="step-3" src="https://github.com/user-attachments/assets/e525cc3e-7433-40eb-8235-892358cea29c" />


<img width="2904" height="1082" alt="step-4" src="https://github.com/user-attachments/assets/532ed95e-9856-481a-9192-e60a95c51cb6" />


### Step 2: Break the Syntax with a Single Quote

Submit a single quote as the category:

```http
GET /filter?category='
```

The response is an **Internal Server Error**, and the error text is unusually revealing: a MongoDB `JSInterpreterFailure` reporting a JavaScript `SyntaxError`, raised inside MongoDB's scripting engine (`mongohelpers.js`) on the server at `127.0.0.1:27017`.

That one response establishes two facts: the input is embedded in a **JavaScript expression** that MongoDB evaluates, and an unmatched quote breaks that expression's syntax. The parameter is being treated as code, not data.

<img width="3024" height="1157" alt="step-5" src="https://github.com/user-attachments/assets/f5a2963d-d058-4ef6-badc-9ab4af122e09" />


<img width="2530" height="776" alt="step-6" src="https://github.com/user-attachments/assets/7fa7dc4d-9b28-4e5c-8652-5dc104ed058d" />


### Step 3: Repair the Syntax with a Valid Payload

Close the string properly so the expression parses again. Using a real category value, append a quote and reopen a harmless continuation (URL-encode the payload with Ctrl-U in Burp):

```text
Gifts'+'
```

This time there is **no syntax error** and the response behaves normally. The quote is no longer "breaking" anything because the payload completes the JavaScript string the server builds — strong evidence of server-side JavaScript injection rather than a generic error.

<img width="2815" height="1016" alt="step-7" src="https://github.com/user-attachments/assets/348209a9-7047-4342-8b10-1bffc33921e9" />


### Step 4: Prove Boolean Control — False Condition

Now inject a complete boolean expression. Keep the category, close the string, and AND the result with false (`0`):

```text
Gifts' && 0 && 'x
```

The query now effectively asks for products in the Gifts category **and false** — so **no products are returned**. The response changed purely because of the injected logic.

<img width="2616" height="840" alt="step-8" src="https://github.com/user-attachments/assets/7d122e3a-1fcb-4000-ae2d-c545d472092e" />

### Step 5: Prove Boolean Control — True Condition

Repeat with a true value (`1`):

```text
Gifts' && 1 && 'x
```

The products in the Gifts category **are returned again**. False hides the data, true restores it: the injected condition is being evaluated by the database exactly as written. Injection is confirmed, with a working boolean oracle.

<img width="2693" height="904" alt="step-9" src="https://github.com/user-attachments/assets/50d7440d-9d19-41a1-850c-6bb4ab8f537d" />


### Step 6: Inject an Always-True Condition to Defeat the Filter

The filter exists to hide unreleased products, which it does by AND-ing its own condition onto the category match. Defeat it with an OR: close the string, then add `|| 1 ||` so the overall expression is true regardless of the other clauses:

```text
Gifts'||1||'
```

The response now contains **all products — including unreleased ones** that the normal filter never shows.

<img width="2602" height="879" alt="step-10" src="https://github.com/user-attachments/assets/9ee25ef8-0b5e-4623-942f-a1cc0386cd71" />


### Step 7: Verify in the Browser

Right-click the response in Burp and choose **Open response in browser** (or copy the request URL into the browser). The rendered page shows the full product list with the previously hidden, unreleased products visible. Viewing them solves the lab.

<img width="2725" height="1320" alt="step-11" src="https://github.com/user-attachments/assets/62225135-1f92-49c4-8216-ad246393c426" />


<img width="3150" height="1498" alt="step-12" src="https://github.com/user-attachments/assets/b44a700c-7863-4986-a275-c28ca4e33b9d" />


## Key Takeaways

- Detection follows a ladder: break it (`'` → JavaScript syntax error), fix it (`Gifts'+'` → normal response), then steer it (false hides data, true restores it). Each step removes a possible innocent explanation.
- Verbose errors were the loudest signal here — the response named MongoDB, its JavaScript interpreter, and the exact failure, confirming both the database technology and the injection mechanism in one shot.
- The final bypass needed no knowledge of the product schema: `|| 1 ||` short-circuits whatever conditions the application added, which is why always-true payloads are the standard first exploit for boolean-based injection of any dialect.
- NoSQL injection is not only about JSON operators (`$ne`, `$gt`); wherever input is concatenated into a JavaScript-evaluated query, classic string-breakout technique applies.

## Remediation

Never build database queries by concatenating user input into JavaScript strings, and avoid `$where` and other server-side script evaluation with user-influenced data entirely — use structured query parameters with typed values instead. Validate input against an allowlist where the value domain is known (here, a fixed set of category names). Apply least privilege to the database account, and suppress detailed database errors in client responses: log them server-side, return a generic error, and treat a JavaScript syntax error from user input as a security event worth alerting on.
