# Excessive Trust in Client-Side Controls

**Platform:** PortSwigger Web Security Academy
**Category:** Business Logic
**Difficulty:** Apprentice
**Status:** ✅ Solved

## Background

Excessive trust in client-side controls means developers assume the user will only ever send data through the interface the website provides, so they rely on client-side validation alone. But an attacker is not limited to the interface: with a tool such as Burp Suite, the request can be intercepted between the browser and the server, modified, and then sent on. If the server performs no proper validation or integrity checks of its own, the attacker can manipulate the application's logic through the modified input. The simple rule: client-side validation is never sufficient for security — important validation must always happen server-side.

## Lab Objective

This lab doesn't adequately validate user input. Exploit a logic flaw in its purchasing workflow to buy items for an unintended price — specifically, buy a "Lightweight l33t leather jacket".

**Provided resources:** your credentials `wiener:peter`, with $100 of store credit. The jacket costs $1,337.

<img width="2193" height="871" alt="step-1" src="https://github.com/user-attachments/assets/3c169e31-7975-47c1-bd2c-6841b78016be" />

## Walkthrough

### Step 1: Compare the Price with Your Credit

Log in and look at the target product. Your account holds $100 in credit, and the jacket costs $1,337 — a legitimate purchase is impossible, by design.

<img width="3145" height="1790" alt="step-2" src="https://github.com/user-attachments/assets/4b8046ac-5c3f-4713-b07c-404d6e78d39b" />

<img width="3004" height="1596" alt="step-3" src="https://github.com/user-attachments/assets/1331b384-5344-4933-9c5b-22129377b3d6" />

<img width="3077" height="1435" alt="step-4" src="https://github.com/user-attachments/assets/2d512822-de38-443b-b218-f551db1ac77d" />

<img width="2552" height="1486" alt="step-5" src="https://github.com/user-attachments/assets/8164b825-67fb-4558-9783-cbf2cf6e1a6f" />

### Step 2: Try to Buy It Anyway

Add the jacket to your cart and place the order. The store refuses with an error: not enough money. In the browser's world, that is the end of the story.

<img width="2237" height="1536" alt="step-6" src="https://github.com/user-attachments/assets/5a59c201-301d-4284-ae21-f917b5af0537" />

### Step 3: Intercept the Cart Request and Rewrite the Price

Repeat the purchase with Burp running and capture the add-to-cart request. The `POST /cart` body contains everything the server uses to price the order — including the price itself, supplied by the client:

```http
POST /cart HTTP/2
Host: <lab host>
Cookie: session=<your session>

productId=1&redir=PRODUCT&quantity=1&price=133700
```

Change the `price` parameter to a nominal value within your means (here, `price=13`) and forward the request. The server accepts the client-supplied price without ever checking it against its own catalog.

<img width="2658" height="1506" alt="step-7" src="https://github.com/user-attachments/assets/e1fe04f4-a1cc-4279-8dce-ea64361adee6" />

<img width="2859" height="1465" alt="step-8" src="https://github.com/user-attachments/assets/422f4328-0334-4e7a-973e-61f54a969b1b" />

<img width="2556" height="1290" alt="step-9" src="https://github.com/user-attachments/assets/ebe2270f-6f07-4b42-8cc5-7d8824fd78af" />

### Step 4: Place the Order

Your cart and balance now reflect the tampered price. Place the order — this time it goes through. You have bought the $1,337 jacket with $100 of credit, and the lab is solved.

<img width="2173" height="1443" alt="step-10" src="https://github.com/user-attachments/assets/5767827e-ceb7-4677-8045-9f504fd2ee4c" />

<img width="3176" height="1259" alt="step-11" src="https://github.com/user-attachments/assets/8e0297d3-675d-478b-af6f-f673c416e600" />

## Key Takeaways

- Anything the browser sends — prices, quantities, discounts, totals — is a suggestion, not a fact. The server must recompute or look up every value that affects money.
- The giveaway was structural: a `price` field in the request body at all. Legitimate designs reference the product (`productId`) and price it server-side; a price parameter exists only because the client was trusted to state it.
- Client-side validation is a user-experience feature (fast feedback), never a security boundary.

## Remediation

Never accept prices or totals from the client. On the server, derive the price from the product ID using server-side catalog data, compute totals there, and validate quantities and discounts against server-held rules. Treat every field of every request as untrusted, no matter how the official interface behaves.
