# Insufficient Workflow Validation

**Platform:** PortSwigger Web Security Academy
**Category:** Business Logic
**Difficulty:** Practitioner
**Status:** ✅ Solved

## Background

Picture a checkout workflow: select a product, provide details, pay, receive confirmation. Developers often assume every user will walk that sequence in order — `1 → 2 → 3 → 4` — and build later steps on the assumption that earlier ones must have succeeded: *"if the user reached the confirmation page, payment must already be complete."*

An attacker is under no obligation to follow the interface. With requests captured in Burp Suite, steps can be repeated, modified, skipped, reordered, or invoked directly — a practice sometimes called **forced browsing**. The frontend's workflow is not a security control: the server must verify, on every critical request, that the required preceding conditions actually hold. Errors provoked by unexpected sequences (null values, missing parameters) are themselves useful clues about what the developer assumed.

## Lab Objective

This lab makes flawed assumptions about the sequence of events in the purchasing workflow. Exploit the flaw to buy a "Lightweight l33t leather jacket".

**Provided resources:** your credentials `wiener:peter`, with $100 of store credit — far less than the jacket's price.

<img width="2217" height="868" alt="step-1" src="https://github.com/user-attachments/assets/a8748ed1-3961-4c07-8f4f-2436844bafa6" />


## Walkthrough

### Step 1: Confirm the Jacket Is Unaffordable

Log in and check your position: $100 of credit against the jacket's price. Add the jacket to the cart and place the order — it fails with a "not enough balance" error, as expected.

<img width="3040" height="1776" alt="step-2" src="https://github.com/user-attachments/assets/ede50548-41bf-48d9-90c3-a4b15eea5466" />


<img width="2272" height="1406" alt="step-3" src="https://github.com/user-attachments/assets/0e18b269-4538-44dd-b9c5-bbf247aa9831" />


<img width="3088" height="1349" alt="step-4" src="https://github.com/user-attachments/assets/55dec54d-9722-487c-ab9d-ec6c06f2112e" />


<img width="3062" height="1815" alt="step-5" src="https://github.com/user-attachments/assets/5c538b16-9887-4637-ae8e-752072a67dac" />


<img width="2886" height="1580" alt="step-6" src="https://github.com/user-attachments/assets/38a93ea8-d035-4173-9dcf-2c9c6370bc81" />


### Step 2: Learn the Workflow from a Cheap Purchase

To see the whole process, buy something inexpensive instead — a low-priced item you *can* afford — and study every request in Burp as the purchase goes through. Pay attention to the final step of the flow, the order-confirmation request:

```http
GET /cart/order-confirmation?order-confirmed=true HTTP/2
Host: <lab host>
Cookie: session=<your session>
Referer: https://<lab host>/product?productId=2
```

Note the `Referer` header: it points back at the product page of the item just bought. The confirmation step appears to identify *what was purchased* from where the request claims to come from — and nothing in the request proves that any payment step succeeded.

<img width="3136" height="1539" alt="step-7" src="https://github.com/user-attachments/assets/ab75555b-562b-40ad-b4ff-ce9f00d98f32" />


<img width="2931" height="1668" alt="step-8" src="https://github.com/user-attachments/assets/17bf7a26-7388-48b5-aabb-c95897de282b" />


<img width="2158" height="1191" alt="step-9" src="https://github.com/user-attachments/assets/d0667d4b-f5a4-41d1-84ef-aa17f651be44" />


<img width="2951" height="1186" alt="step-10" src="https://github.com/user-attachments/assets/05acdafd-1e60-40bc-86cf-41df1dd1bd0e" />


### Step 3: Invoke the Confirmation Step Directly, for the Jacket

Now put the jacket in your cart (its checkout will fail on balance — that no longer matters) and craft the confirmation request yourself, changing exactly two things relative to the genuine pattern:

1. Request the confirmation URL directly: `GET /cart/order-confirmation?order-confirmed=true`
2. Set the `Referer` header to the jacket's product page: `/product?productId=1`

```http
GET /cart/order-confirmation?order-confirmed=true HTTP/2
Host: <lab host>
Cookie: session=<your session>
Referer: https://<lab host>/product?productId=1
```

Send it. The server returns the order-confirmation page — the response even carries the lab's solved state. The confirmation step never independently verified that the jacket's checkout had succeeded or that its price had been paid; it trusted the workflow sequence and the `Referer` header to vouch for both.

<img width="2220" height="1129" alt="step-11" src="https://github.com/user-attachments/assets/3049266d-b925-4ab3-8a9f-1d9b08e3c7ad" />


<img width="2930" height="1255" alt="step-12" src="https://github.com/user-attachments/assets/c11a09dd-ff4d-4b1a-bfd6-655ef80ed314" />


<img width="3631" height="1218" alt="step-13" src="https://github.com/user-attachments/assets/48460454-cdae-4f0f-a782-7649126808d9" />


## Key Takeaways

- Never infer state from sequence. "The user is asking for confirmation" is not evidence that "the user paid" — each step must check the recorded state server-side.
- Metadata like the `Referer` header is attacker-supplied. Using it to identify what was purchased handed the attacker the choice of what to be confirmed for.
- Map the full workflow with a legitimate run first (here, the cheap purchase), then attack the seams: skip steps, replay the final step, and swap the context each step relies on.

## Remediation

Track workflow state server-side: an order should only be confirmable if a server-held record shows payment completed for that exact order and cart contents. Each step should validate its preconditions against that record and refuse out-of-sequence invocation. Never derive security-relevant facts (what was bought, whether it was paid for) from client-controlled headers or parameters.
