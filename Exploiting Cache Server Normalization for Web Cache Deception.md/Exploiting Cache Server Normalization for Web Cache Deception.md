# Exploiting Cache Server Normalization for Web Cache Deception

**Platform:** PortSwigger Web Security Academy
**Category:** Web Cache Deception
**Difficulty:** Practitioner
**Status:** ✅ Solved

## Background

This lab is the mirror image of origin-side normalization. Here it is the **cache** that normalizes the URL before making its decision: it URL-decodes the path and resolves dot-segments, then applies its cache rules to the *normalized* result. The origin server does not do this — it routes the request on the path much more literally, honouring encoded delimiters such as `%23` (an encoded `#`) by treating everything from that point on as outside the path. So a single URL can mean "/my-account" to the origin and "/resources" to the cache at the same time. If the normalized form lands under a static cache rule, the private response the origin returns is stored under a key the attacker controls — web cache deception again, with the roles of the two components reversed.

## Lab Objective

The application is vulnerable to web cache deception because of a discrepancy in how the cache and the origin server normalize URL paths. Exploit the vulnerability to steal the victim user `carlos`'s API key, and submit it via the lab banner.

**Provided resources:** your account `wiener:peter`, the victim user `carlos`, an exploit server, and a list of candidate delimiter characters supplied with the lab.

<img width="2173" height="1255" alt="step-1" src="https://github.com/user-attachments/assets/7c79c763-6052-4a57-b448-ba09fd80067a" />

<img width="3226" height="1615" alt="step-2" src="https://github.com/user-attachments/assets/d989f339-c36c-4432-8378-0fdf55bb4ab2" />

## Walkthrough

### Step 1: Identify the Sensitive Page

Log in as `wiener` and open **My Account** — the page displays your API key, and `carlos`'s copy of it is the target.

<img width="3080" height="1471" alt="step-3" src="https://github.com/user-attachments/assets/5b830f5a-bef8-46a2-851a-494781ebb37d" />

Analyse the `GET /my-account` request in Burp and send it to Intruder.

<img width="3314" height="1251" alt="step-4" src="https://github.com/user-attachments/assets/2cc6693d-36b9-45bb-b6a3-5f8b1c90900e" />

### Step 2: Fuzz Delimiters — Including the Encoded Ones

The lab supplies a list of characters that may be used as delimiters. Load it as the Intruder payload set, with the payload position after `/my-account` and URL-encoding disabled.

<img width="2246" height="1440" alt="step-5" src="https://github.com/user-attachments/assets/4e8b3f61-c924-47d6-a460-cfd8df5370ce" />

<img width="3752" height="1706" alt="step-6" src="https://github.com/user-attachments/assets/3009005d-2fd2-4e3b-abea-973080b60eb6" />

Four payloads return **200**: `#`, `?`, `%23` and `%3F`. The origin accepts both the literal delimiters and their percent-encoded forms — a first hint that the two sides decode differently.

<img width="1990" height="721" alt="step-7" src="https://github.com/user-attachments/assets/76c03246-7d87-478f-95d4-ef9a53496e75" />

Testing the obvious combinations disappoints, however: `/my-account?abc.js` shows no evidence of caching, and neither do the `%23` and `%3f` variants with a `.js` extension. There is no extension-based rule to abuse here.

### Step 3: Test Origin Normalization — It Isn't There

Try the dot-segment route against the origin:

```http
GET /aaa/..%2fmy-account
```

The response is **404**. Unlike the previous lab, this origin does **not** decode or resolve dot-segments — it will not be talked into serving `/my-account` by a mangled path.

<img width="2416" height="1038" alt="step-8" src="https://github.com/user-attachments/assets/736d1aab-5c87-4970-bccf-226e22e57641" />

### Step 4: Test Cache Normalization — It Is There

Static resources on the site share the `/resources` prefix, and their responses show caching. Send one of those requests to Repeater and prepend an arbitrary directory plus an encoded dot-segment:

```http
GET /aaa/..%2fresources/YOUR-RESOURCE
```

The 404 response carries `X-Cache: miss`; resend it and the header becomes `hit`. The cache decoded `%2f`, resolved `/aaa/..` away, saw a path under `/resources`, and applied its static directory rule. The cache normalizes — the origin does not.

<img width="2777" height="1175" alt="step-9" src="https://github.com/user-attachments/assets/ab0a464e-b89b-47cb-917c-d970ff31122b" />

<img width="2922" height="797" alt="step-10" src="https://github.com/user-attachments/assets/45e1c942-225a-4d35-94ee-b97545999094" />

<img width="2700" height="828" alt="step-11" src="https://github.com/user-attachments/assets/ca130bcc-5eca-4099-80c7-68291ea469b5" />

(Further confirmation tests are possible — for example, a dot-segment *after* the prefix, `/resources/..%2fYOUR-RESOURCE`, does not produce the same effect — but the miss → hit pair on the prepended path is already the signature needed.)

### Step 5: Build the Two-Faced URL

Combine both behaviours in one path. The origin must see `/my-account`; the cache, after decoding and resolving, must see a path under `/resources`. Start from the `?` delimiter form:

```http
GET /my-account?%2f%2e%2e%2fresources
```

This returns 200 with the API key — but no evidence of caching, because the cache also honours `?` and never looks inside the query string.

<img width="2976" height="985" alt="step-12" src="https://github.com/user-attachments/assets/82c2719a-1b53-4d7f-997f-618135ef04f3" />

Swap the literal `?` for its encoded form, `%23`:

```http
GET /my-account%23%2f%2e%2e%2fresources
```

Now everything lines up: **200 with the API key, and `X-Cache: miss`** — resend, and it becomes `hit`. The origin treats `%23` as a delimiter and serves the account page; the cache decodes it, resolves `%2e%2e%2f` as `../`, reduces the path to `/resources`, and caches the response under its static rule.

<img width="2544" height="746" alt="step-13" src="https://github.com/user-attachments/assets/c2835d62-3f85-43a5-893a-1e1290b4245c" />

<img width="2453" height="665" alt="step-14" src="https://github.com/user-attachments/assets/6f75c992-ecc6-4597-a1ac-5ac0fb7da93d" />

### Step 6: Deliver the Trap to the Victim

Open the exploit server and craft an exploit navigating the victim to the same URL, with an arbitrary query parameter (`?wcd`) added as a cache buster so the victim's response is stored under a fresh key:

```html
<script>document.location="https://YOUR-LAB-ID.web-security-academy.net/my-account%23%2F..%2Fresources?wcd"</script>
```

<img width="3090" height="1786" alt="step-15" src="https://github.com/user-attachments/assets/e86e44cc-8afb-4c2e-adca-3accf0191e61" />

Click **Deliver exploit to victim**, then visit the delivered URL yourself. The cache serves the stored response: `carlos`'s account page, with his API key.

<img width="2583" height="1495" alt="step-16" src="https://github.com/user-attachments/assets/834cbd64-8984-40bd-b94b-ae1ed9570fb4" />

Submit the key using the button in the lab banner. The lab is solved.

<img width="3096" height="991" alt="step-17" src="https://github.com/user-attachments/assets/8b499a5f-6515-436e-bbc4-3d643476bc4d" />

<img width="3169" height="1133" alt="step-18" src="https://github.com/user-attachments/assets/fa394da8-c3b1-4260-b965-6f1060ea005a" />

## Key Takeaways

- Normalization discrepancies cut both ways. In the previous lab the origin normalized and the cache did not; here the cache normalizes and the origin does not. The testing method is identical — only the direction of the mismatch changes.
- Encoded delimiters (`%23`, `%3F`) are first-class test cases, not edge cases: a component that decodes before parsing and one that parses before decoding will route the same bytes to different resources.
- Dead ends are data. The failed `?` and `.js` tests, and the origin's 404 on the dot-segment path, are what narrowed the search to the one combination that works: encoded delimiter + encoded dot-segment + static prefix.
- The final URL is a small sentence in two languages at once: "account page" in the origin's grammar, "static resource" in the cache's.

## Remediation

Cache and origin must normalize identically — ideally, normalize once at the edge and pass only the normalized path inward, so no component ever re-interprets raw encodings or dot-segments differently. A cache should not decode or resolve a path into a cacheable prefix when the origin will route the un-normalized form elsewhere; prefix and extension rules should be evaluated against exactly the path the origin will serve. As always, authenticated responses should carry `Cache-Control: private, no-store`, which removes the impact even when a parsing discrepancy slips through.
