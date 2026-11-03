# Flawed Enforcement of Business Rules

**Platform:** PortSwigger Web Security Academy
**Category:** Business Logic
**Difficulty:** Apprentice
**Status:** ✅ Solved

## Background

The business rule sounds simple: *each discount code can be applied only once per order.* The enforcement, however, only checks whether the code you are applying is the same as the one applied **immediately before it**. Apply code A twice in a row and the second attempt is refused — but slip code B in between, and code A is welcome again. The rule was implemented as a check on the last action instead of on the order's history, so with two different codes, an attacker can alternate them indefinitely and stack discounts the store never intended to combine more than once.

## Lab Objective

This lab has a logic flaw in its purchasing workflow. Exploit the flaw to buy a "Lightweight l33t leather jacket".

**Provided resources:** your credentials `wiener:peter`, with $100 of store credit. The jacket costs $1,337.

<img width="2183" height="837" alt="step-1" src="https://github.com/user-attachments/assets/1d5bdc8e-bb94-4492-acd5-d75021ae7bcb" />




## Walkthrough

### Step 1: Collect the First Code and Check Your Balance

The store advertises a new-customer discount right on the homepage: `NEWCUST5`, worth $5 off at checkout. Log in and confirm your position — $100 of credit against a $1,337 jacket. Codes will have to do the heavy lifting.

<img width="3059" height="1820" alt="step-2" src="https://github.com/user-attachments/assets/8dabfd65-0071-4e38-abc7-87c669fdf986" />
<img width="2330" height="1612" alt="step-3" src="https://github.com/user-attachments/assets/451945e3-4327-443f-985a-2501ab3207f8" />
<img width="3188" height="1229" alt="step-4" src="https://github.com/user-attachments/assets/a4334556-4580-4b9e-a264-7e57bb568d41" />

### Step 2: Probe How the Coupon Rule Is Enforced

Test the mechanics on something cheap first: add a $5.44 item to the cart and apply `NEWCUST5`. The discount lands — the price drops by $5 to $0.44. Apply the same code a second time and the store refuses: "coupon already applied." So a code genuinely can't be used twice… at least, not back-to-back.

<img width="3022" height="1794" alt="step-5" src="https://github.com/user-attachments/assets/fa6c0788-66fb-4811-a756-436f16ebc58c" />
<img width="2029" height="1499" alt="step-6" src="https://github.com/user-attachments/assets/4fcad4b4-44df-4946-a571-17df0e1c584a" />

### Step 3: Find a Second Code

Studying the purchase requests in Burp doesn't open a direct route, so look at what else the site offers. A newsletter signup form catches the eye — and signing up earns a second discount code: `SIGNUP30`, worth 30% off. Now you hold two different codes.

<img width="3115" height="955" alt="step-7" src="https://github.com/user-attachments/assets/b45bf291-2d07-4712-869a-8daeb39ff3ce" />
<img width="2947" height="1119" alt="step-8" src="https://github.com/user-attachments/assets/d77e41b0-4b99-4036-9ba3-794785acd7a6" />
<img width="2834" height="873" alt="step-9" src="https://github.com/user-attachments/assets/ff3c6e33-55ac-4774-a490-6b4def94acdd" />

### Step 4: Alternate the Codes Until the Jacket Is Affordable

Add the jacket to the cart and apply the codes — never the same one twice in a row, always alternating:

```text
NEWCUST5 → SIGNUP30 → NEWCUST5 → SIGNUP30 → …
```

Each application passes the check, because the code being applied always differs from the one applied just before it. `NEWCUST5` chips off $5 at a time, and every `SIGNUP30` takes another 30% off the *current* total, so the price collapses quickly: $1,337 falls through the hundreds and keeps dropping. Keep alternating until the total falls below your $100 credit, then place the order. The purchase succeeds — the lab is solved.

<img width="3263" height="1696" alt="step-10" src="https://github.com/user-attachments/assets/aad51b7f-0960-46b9-ba64-14a4f8fd2825" />

<img width="2320" height="1679" alt="step-11" src="https://github.com/user-attachments/assets/93300a4f-1d1f-40c0-87ff-87a80b69d097" />
<img width="2025" height="1640" alt="step-12" src="https://github.com/user-attachments/assets/83807bc3-01e5-4a78-8a6a-2d8f114084b9" />
<img width="3264" height="1268" alt="step-13" src="https://github.com/user-attachments/assets/04974e34-bebd-4e56-90d6-cc68e36c488c" />

## Key Takeaways

- Enforce rules against the full history they govern, not just the most recent event. "Different from the last one" is not the same as "not used before."
- Discount *stacking* policy is a business rule too: even legitimate one-time codes become an exploit when they can be combined and repeated without an overall cap.
- A second code changed everything — when probing coupon logic, inventory every code the application issues (banners, newsletters, emails) before judging what is possible.

## Remediation

Record every code applied to an order server-side and reject any code that already appears in that record, regardless of order or interleaving. Decide and enforce an explicit stacking policy (which codes may combine, and a maximum total discount), and re-validate the final price against that policy at checkout.



