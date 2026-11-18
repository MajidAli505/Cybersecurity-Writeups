# Blind XXE with Out-of-Band Interaction

**Platform:** PortSwigger Web Security Academy
**Category:** XXE Injection
**Difficulty:** Apprentice
**Status:** ✅ Solved

## Background

In the earlier XXE labs the application reflected the parsed value back, so files and SSRF responses arrived inside the HTTP response. That reflection is a convenience, not a requirement. When the parser still resolves external entities but nothing it reads is ever shown to the user, the vulnerability is **blind** — the injection works exactly the same, but the result has to be observed somewhere else. The standard observation post is an **out-of-band (OAST)** server the attacker controls: if the target application can be made to resolve a unique attacker hostname, a DNS lookup or HTTP request arriving at that server proves the entity was fetched, even though the application's own response says nothing.

## Lab Objective

The application uses the XML-based stock checker and is vulnerable to blind XXE. Exploit the vulnerability so that the application triggers an out-of-band interaction with an attacker-controlled server (Burp Collaborator), confirming the blind XXE.

**Provided resources:** the lab shop application and Burp Suite's Collaborator.

<img width="2189" height="821" alt="step-1" src="https://github.com/user-attachments/assets/afd1541f-12e7-40a8-a9d7-b9fffe717adb" />


## Walkthrough

### Step 1: Find the XML Input

Open a product page, use **Check stock**, and capture the `POST /product/stock` request — the same XML `stockCheck` body as in the previous labs. Send it to Burp Repeater.

<img width="3021" height="1739" alt="step-2" src="https://github.com/user-attachments/assets/221eefbd-bd40-4eab-bbcc-911f7511d066" />


<img width="2824" height="937" alt="step-3" src="https://github.com/user-attachments/assets/d27a5cc4-f98e-4d4e-aea2-02fbc2368d38" />


<img width="2720" height="1351" alt="step-4" src="https://github.com/user-attachments/assets/83933f61-5d45-4690-b1a8-709f1b50f6a3" />


### Step 2: Generate a Collaborator Payload

In Burp, open the **Collaborator** tab and click **Copy to clipboard** to generate a unique payload hostname (an `*.oastify.com` subdomain that belongs to this Collaborator session only).

<img width="2595" height="1109" alt="step-5" src="https://github.com/user-attachments/assets/2e69dba5-02b0-4c5f-b451-575a799ad7d6" />


### Step 3: Inject the Entity Pointing at Collaborator

Define the external entity with the Collaborator URL as its target and reference it in `productId`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "http://YOUR-COLLABORATOR-PAYLOAD.oastify.com"> ]>
<stockCheck>
    <productId>&xxe;</productId>
    <storeId>1</storeId>
</stockCheck>
```

Send the request. The response is an ordinary stock count — completely normal-looking, with nothing of the entity's content in it. That silence is what "blind" means; it is not evidence of failure.

<img width="1810" height="1235" alt="step-6" src="https://github.com/user-attachments/assets/45153bbb-e5d0-459c-83bf-a71a7aa124b1" />


<img width="2746" height="1421" alt="step-7" src="https://github.com/user-attachments/assets/263be759-8567-40fe-b0af-8778ad11c07d" />


### Step 4: Poll Collaborator and Observe the Interaction

Return to the Collaborator tab and click **Poll now**. Four interactions are listed for the payload: **two DNS lookups and two HTTP requests**, originating from the lab server's infrastructure.

That is the complete proof chain. The DNS queries show the application resolved the attacker hostname; the HTTP requests show its XML parser actually fetched the entity URL. The application processed the external entity — the XXE is real, and it is blind because none of it was visible in the application's response. Triggering this out-of-band interaction solves the lab.

<img width="2348" height="852" alt="step-8" src="https://github.com/user-attachments/assets/ff5be34f-bdf3-4d5a-bf36-357416f61c6b" />


<img width="3075" height="1287" alt="step-9" src="https://github.com/user-attachments/assets/1f546b6c-732b-4223-a630-f77f20e62e9d" />


## Key Takeaways

- "No output in the response" does not mean "not vulnerable". Blind XXE is confirmed by side effects — DNS and HTTP interactions on infrastructure the attacker watches.
- The unique Collaborator subdomain is what makes the signal trustworthy: only this test payload exists, so any interaction with it must have come from this injection.
- DNS interactions alone already prove server-side resolution (and can leak data via subdomains in later attacks); the HTTP interaction additionally proves the parser fetched the resource over HTTP.
- Detection workflow for any suspected blind injection: baseline request → entity pointing at a fresh OAST payload → poll → correlate by timestamp and payload ID.

## Remediation

Disable DTDs and external entities in the XML parser — blind XXE disappears with the same fix as the reflected variant, because the parser simply never makes the outbound request. Egress filtering on application servers (no arbitrary outbound DNS/HTTP to the internet) blunts OAST channels and SSRF generally. Defenders should also monitor for outbound DNS/HTTP requests from application processes to unusual domains: in this attack, the only visible trace of the exploitation lived on the network, not in the application's responses.
