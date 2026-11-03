# Infinite Money Logic Flaw

**Platform:** PortSwigger Web Security Academy
**Category:** Business Logic
**Difficulty:** Practitioner
**Status:** ✅ Solved

## Background

Two of this store's features are each reasonable in isolation: a percentage discount coupon can be applied at checkout, and gift cards can be bought and later redeemed for their **full face value** in store credit. The flaw is in the combination. Nothing stops you buying a gift card *with* the discount and then redeeming it at face value — so a $10 gift card bought for $7 redeems for $10, manufacturing $3 of credit out of nothing. The loop has no natural limit: repeat it enough times and the credit grows without bound. Infinite money.

## Lab Objective

This lab has a logic flaw in its purchasing workflow. Exploit the flaw to buy a "Lightweight l33t leather jacket".

**Provided resources:** your credentials `wiener:peter`, with $100 of store credit. The jacket costs $1,337.

<img width="2199" height="912" alt="step-1" src="https://github.com/user-attachments/assets/5db407fd-4d59-4fcc-b03a-37824f654d0b" />


## Walkthrough

### Step 1: Get the Discount Code

Log in (your balance is $100) and sign up for the store's newsletter, which earns you the coupon code `SIGNUP30` — 30% off at checkout. Note also that the store sells **$10 gift cards**, redeemable from the My account page.

<img width="3205" height="1797" alt="step-2" src="https://github.com/user-attachments/assets/2e3c1223-8ad3-4a8c-98c6-0043ac2125f2" />


<img width="2583" height="1033" alt="step-3" src="https://github.com/user-attachments/assets/48a02ae8-002e-4d5b-921e-ea252b33adcd" />


<img width="2405" height="1649" alt="step-4" src="https://github.com/user-attachments/assets/b299819d-e307-488c-9b23-340423697131" />


<img width="3167" height="903" alt="step-5" src="https://github.com/user-attachments/assets/b791373b-a295-4b71-bde5-b2374a3c74a7" />


<img width="3035" height="1467" alt="step-6" src="https://github.com/user-attachments/assets/9b331420-ab08-4e0b-bd80-2eca619617f8" />


### Step 2: Prove the Loop by Hand

Run the cycle once manually:

1. Add a $10 gift card to your cart and check out with `SIGNUP30` applied. You pay **$7** — your balance drops to $93.
2. Copy the gift card code from the order confirmation.
3. On the My account page, redeem the code. **$10** of credit lands in your account — your balance is now $103.

One loop: **+$3**, created from nothing. Your balance is higher than when you started, and nothing in the flow prevents doing it again.

<img width="2015" height="1622" alt="step-7" src="https://github.com/user-attachments/assets/9158fdd0-4d08-4579-ad03-c270d4b7ae5e" />


<img width="1487" height="1083" alt="step-8" src="https://github.com/user-attachments/assets/484eb984-1cfe-4c94-9d9a-4a263551e519" />


<img width="2973" height="1647" alt="step-9" src="https://github.com/user-attachments/assets/9e1b9867-f778-4ac2-8499-4c41953b9187" />


<img width="2037" height="1618" alt="step-10" src="https://github.com/user-attachments/assets/8712d2f4-819d-4ce1-9da9-0c6fd15493fa" />
<img width="2311" height="1609" alt="step-11" src="https://github.com/user-attachments/assets/3b5131a7-c0a7-4c69-b423-5214497979b5" />


### Step 3: Study the Requests

In Burp's proxy history, the redemption is a single request — `POST /gift-card` with the code in the `gift-card` parameter. The whole loop is five requests: add to cart, apply coupon, check out, view the order confirmation (whose response contains the new gift card code), and redeem. That makes it a perfect candidate for a Burp **macro** driven by a **session handling rule**: every time a chosen request fires, Burp replays the whole buy-and-redeem cycle automatically.
<img width="2104" height="1197" alt="step-12" src="https://github.com/user-attachments/assets/85b30e3b-1cbb-422f-b7fb-11d06f2e70b4" />


### Step 4: Build the Automation (Session Rule + Macro)

Open Settings → Sessions and add a session handling rule. Set its URL scope to include all URLs, and give it a "Run a macro" action. In the Macro Recorder, select the five requests of the loop in order:

```text
POST /cart
POST /cart/coupon
POST /cart/checkout
GET  /cart/order-confirmation?order-confirmed=true
POST /gift-card
```

Two pieces of glue make the macro self-sustaining:

1. **Extract:** configure the order-confirmation item to define a custom parameter named `gift-card`, taken from the gift card code highlighted in that response.
2. **Inject:** configure the `POST /gift-card` item so its `gift-card` parameter is derived from that prior response (response 4).

Test the macro: the code generated in the confirmation response matches the code sent in the redemption request, and the redemption returns a 302. One trigger request now completes a full money loop on its own.
<img width="3826" height="1404" alt="step-13" src="https://github.com/user-attachments/assets/31223b0a-20fd-4875-b8aa-ccdd30223db0" />

<img width="2788" height="1749" alt="step-14" src="https://github.com/user-attachments/assets/74ed58d9-c18c-4053-a795-2f3fa959321f" />


<img width="1606" height="1156" alt="step-15" src="https://github.com/user-attachments/assets/90d557c8-d5bd-486f-bbb5-a5b2035c4ce8" />

<img width="1680" height="1382" alt="step-16" src="https://github.com/user-attachments/assets/ef29a6f2-59b5-4e69-a906-9387fcf7d5f4" />

<img width="1940" height="1300" alt="step-17" src="https://github.com/user-attachments/assets/98858518-0eb2-4105-bac1-a9292fda065c" />


<img width="2511" height="1673" alt="step-18" src="https://github.com/user-attachments/assets/f6bd616f-d8d5-4823-875a-dc1ba338e0fd" />


<img width="2090" height="1948" alt="step-19" src="https://github.com/user-attachments/assets/0db52359-18ee-4766-add4-62009a720d53" />


<img width="3350" height="1703" alt="step-20" src="https://github.com/user-attachments/assets/0936d847-e884-43c4-9af0-c6816428b340" />


<img width="3160" height="1642" alt="step-21" src="https://github.com/user-attachments/assets/bab05509-6262-4878-bbac-c2dfb65a84ae" />


<img width="2811" height="1768" alt="step-22" src="https://github.com/user-attachments/assets/d5a992c4-6c9d-4060-983f-a842fac8dbf5" />


<img width="2307" height="2048" alt="step-23" src="https://github.com/user-attachments/assets/38232a88-bbee-4e65-8642-dc83cc7fb99a" />


<img width="2415" height="1723" alt="step-24" src="https://github.com/user-attachments/assets/4b80b8b7-e939-4927-818c-67d42cc1a30f" />


<img width="2910" height="1585" alt="step-25" src="https://github.com/user-attachments/assets/53a9d076-f5b2-453d-9604-7310e0629dc8" />


<img width="2747" height="1782" alt="step-26" src="https://github.com/user-attachments/assets/6d258d5f-eeb8-4000-86bf-8eebfb582777" />


<img width="2667" height="1770" alt="step-27" src="https://github.com/user-attachments/assets/46b85250-9d69-45e5-8a3f-e075d53da768" />


<img width="2775" height="1573" alt="step-28" src="https://github.com/user-attachments/assets/9ad7fa8c-eed6-4e53-b48d-0a3498ad2271" />


### Step 5: Fire the Loop 412 Times

The session rule runs the macro whenever a request in its scope is made — so any request can be the trigger. Send `GET /my-account` to Burp Intruder, choose a Sniper attack with **null payloads generating 412 requests**, and put the attack in a resource pool with **maximum concurrent requests = 1** so the loops run strictly one after another. Start the attack and let it finish: 412 loops × $3 ≈ $1,236 of manufactured credit.

<img width="2415" height="990" alt="step-29" src="https://github.com/user-attachments/assets/0a6b35bb-95f6-4574-9eba-ee34ffd0967c" />


<img width="2347" height="1705" alt="step-30" src="https://github.com/user-attachments/assets/6b55d906-b951-490a-a164-539faa739f80" />


<img width="3772" height="1702" alt="step-31" src="https://github.com/user-attachments/assets/8def992a-8357-4986-91ae-6c6873c3ca5b" />


<img width="3828" height="1353" alt="step-32" src="https://github.com/user-attachments/assets/57c2cf4d-fe26-48d0-a1c1-87d1061f8456" />


### Step 6: Buy the Jacket

Check your account: the credit has grown far beyond the starting $100. Buy the jacket, applying `SIGNUP30` once more for good measure — the final order totals **$935.90**, paid from the manufactured credit, with $398.10 still left over. The lab is solved.

<img width="2957" height="1990" alt="step-33" src="https://github.com/user-attachments/assets/4ed1e5cf-dbac-4fec-9039-d4d731b6c13e" />


<img width="2231" height="1091" alt="step-34" src="https://github.com/user-attachments/assets/d83aa306-c1a8-4e72-8dd9-6ef0793aab5f" />


<img width="3097" height="1291" alt="step-35" src="https://github.com/user-attachments/assets/bc338825-887b-4399-bf94-a93fd9ae851d" />


## Key Takeaways

- The vulnerability was an *interaction*: discounting and gift-card redemption were each "working as designed," and jointly they were a money printer. Reviewing features one at a time misses flaws that live between them.
- Face value is a promise: any instrument redeemable at face value must never be sellable below it, or the difference is free money, repeatable at machine speed.
- The exploitation craft here is automation discipline — a session rule plus a macro that extracts a value from one response and injects it into a later request, sequenced strictly (concurrency 1) so the state machine stays coherent.

## Remediation

Exclude gift cards (and other cash-like instruments) from percentage discounts, or redeem discounted gift cards at their purchase price rather than face value. Enforce it server-side at both purchase and redemption, and monitor for repeated buy-redeem cycles, which have no legitimate high-volume use.
