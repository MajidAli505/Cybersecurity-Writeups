# Low-Level Logic Flaw

**Platform:** PortSwigger Web Security Academy
**Category:** Business Logic
**Difficulty:** Practitioner
**Status:** ✅ Solved

## Background

A low-level logic flaw lives in the machinery underneath the business rules — here, in how the back end represents money. The store keeps prices in **cents** and accumulates the cart total in a **32-bit signed integer**. That type has a hard ceiling:

```text
Maximum:   2,147,483,647
Minimum:  -2,147,483,648
```

When arithmetic pushes a value one past the maximum, it does not stop or error — it **wraps around** to the minimum and starts counting up again. That is integer overflow. If an attacker can keep adding to the total, they can push it over the ceiling, through the negative range, and park it anywhere they like — including a small positive number that a $100 balance can pay.

## Lab Objective

This lab doesn't adequately validate user input. Exploit a logic flaw in its purchasing workflow to buy items for an unintended price — specifically, buy a "Lightweight l33t leather jacket".

**Provided resources:** your credentials `wiener:peter`, with $100 of store credit. The jacket costs $1,337 — 133,700 cents.

<img width="2184" height="1006" alt="step-1" src="https://github.com/user-attachments/assets/c4a4c138-0ad8-4bae-b404-c97f4e7c3b89" />

## Walkthrough

### Step 1: Establish the Baseline

Log in, add the jacket to your cart, and try to place the order. It fails: your credit is far below the jacket's price. A direct purchase is impossible.

<img width="3048" height="1526" alt="step-2" src="https://github.com/user-attachments/assets/3d2a0f01-0c63-42cf-81a1-3be89b672a25" />

<img width="2883" height="1383" alt="step-3" src="https://github.com/user-attachments/assets/c799804e-7252-400d-9573-ef90d54010d3" />

<img width="3301" height="1512" alt="step-4" src="https://github.com/user-attachments/assets/dc711680-273f-4ae2-a75e-8ea25bcc672b" />

<img width="2476" height="1286" alt="step-5" src="https://github.com/user-attachments/assets/3db8fe1a-5cb6-488d-8d99-c57244d0f87c" />

### Step 2: Find the Quantity Limit

Capture the `POST /cart` request, send it to Burp Repeater, and probe the `quantity` parameter:

- `quantity=99` is accepted.
- `quantity=100` returns an error.

So the quantity field only allows two digits — at most 99 jackets per request. That still leaves a big lever: each maximal request adds 99 × 133,700 = **13,236,300 cents** to the cart total.

<img width="2466" height="1489" alt="step-6" src="https://github.com/user-attachments/assets/54f98d91-ee12-4c8f-aa89-bc2d9bac018b" />

<img width="2354" height="1424" alt="step-7" src="https://github.com/user-attachments/assets/fae9679f-32dd-48e6-8aae-19ac5c8d67e9" />

<img width="2553" height="1368" alt="step-8" src="https://github.com/user-attachments/assets/e3712275-c575-43c6-90aa-a6bd64b1baab" />

### Step 3: Prove the Overflow with Intruder

Send the cart request to Burp Intruder with the quantity fixed at 99. Configure the attack to repeat the identical request indefinitely (Intruder's "null payloads" option, set to continue indefinitely — the payload never changes; only the repetition matters), and set it running with a single concurrent request.

Watch the cart total as the requests accumulate: it climbs steadily — and then suddenly flips to a **large negative number** and starts counting up toward zero. The total has passed 2,147,483,647 and wrapped around. Overflow confirmed. Stop the attack.

<img width="2668" height="1406" alt="step-9" src="https://github.com/user-attachments/assets/6fdb78b3-bf7e-416d-bcbb-a105c360bd68" />

<img width="3726" height="1483" alt="step-10" src="https://github.com/user-attachments/assets/6caff821-1f64-4b2e-b057-31b29e30a965" />

<img width="2337" height="1404" alt="step-11" src="https://github.com/user-attachments/assets/e01f82d9-7ebf-4e4c-b606-84f5f8ce24b3" />

### Step 4: Aim the Wrap-Around

Clear the cart. Now the goal is precise: land the final total between $0 and $100, which the jacket alone cannot do — its price in cents doesn't divide the wrap-around range evenly. The arithmetic: 323 requests of 99 jackets bring the accumulated total just past the overflow point into negative territory; one further request for 47 jackets leaves the order total at **−$1,221.96** — negative, but close to zero.

So: recreate the Intruder attack, this time generating exactly **323 payloads** (still 99 jackets per request), run it in a resource pool with maximum concurrent requests set to 1 so the requests land strictly in order, and let it finish. Then send one Repeater request adding 47 more jackets.

<img width="3763" height="1470" alt="step-12" src="https://github.com/user-attachments/assets/72338ba0-424d-4ff5-ab03-e1fc5145d6b2" />

<img width="3795" height="1337" alt="step-13" src="https://github.com/user-attachments/assets/fbdfae25-30b2-454b-b285-fb80e969ad88" />

<img width="3174" height="1770" alt="step-14" src="https://github.com/user-attachments/assets/3d7f0004-0131-44ae-8482-ceda849888cd" />

<img width="2562" height="1379" alt="step-15" src="https://github.com/user-attachments/assets/ad2ba804-12b2-4c65-b6c9-d0baac620a12" />

<img width="2607" height="1461" alt="step-16" src="https://github.com/user-attachments/assets/ac061d6c-c931-42f7-9250-6f75b33288b6" />

### Step 5: Balance Back into Range and Check Out

Use Repeater to add a suitable quantity of another, cheaper item so the negative total climbs back into the $0–$100 window. In the final order, 13 × "Dancing In The Dark" ($97.48 each) alongside the jackets brings the total to **$45.28**. Place the order: it succeeds, leaving $54.72 of credit — and the lab is solved.

<img width="3217" height="1576" alt="step-17" src="https://github.com/user-attachments/assets/1e60802a-a2ef-4d2b-9015-ab7f27845d75" />

<img width="2874" height="1642" alt="step-18" src="https://github.com/user-attachments/assets/722dc600-4fa7-4d08-b4ca-985aaf5fa6a2" />

<img width="3089" height="1254" alt="step-19" src="https://github.com/user-attachments/assets/23a36b38-e5be-4b07-ae1c-6c410d3e6b72" />

## Key Takeaways

- Integer types are finite. Any total an attacker can grow without bound is an overflow waiting for enough repetitions.
- Input limits can create a false sense of safety: capping quantity at 99 per request did nothing, because nothing capped the *number of requests* or the accumulated total.
- Overflow exploitation is arithmetic, not luck: know the price in the stored unit (cents), the per-request maximum, and the type's range, and the required request count is a calculation, not a guess.

## Remediation

Use integer types (or decimal types) sized for the real range of the values involved, and check for overflow in money calculations — or use arbitrary-precision arithmetic for currency. Validate cumulative state, not just per-request input: cap total cart value and item counts server-side, and reject totals outside sane business bounds (a negative or wrapped order total should be impossible to reach checkout).
