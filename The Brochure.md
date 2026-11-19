# The Brochure

**Platform:** TryHackMe
**Category:** Challenge Room (OSINT)
**Difficulty:** Easy
**Status:** ✅ Solved (flag captured)

## Background

The Brochure is an **OSINT** room that starts from paper: the task file is a hotel brochure, and it contains **three clues** — names and references that only become useful online. The hotel itself has a social media account; the account's **following list** is the pivot, because among the accounts it follows is a person named in the brochure, **Vera**. Her profile carries **three Base64-encoded strings** — meaningless apart, but joined together and decoded they produce the flag. Like its companion room *Overheard at Breakfast*, the whole investigation is correlation: a printed name → a business account → a person's account → an encoded secret hiding in plain sight.

## Lab Objective

Extract the brochure's three clues, find the hotel's social media account, pivot through its following list to Vera's account, and combine and decode her three Base64 strings to recover the flag.

**Provided resources:** one task file — the hotel brochure.

<img width="3358" height="996" alt="step-1" src="https://github.com/user-attachments/assets/f74f30dc-1e24-4141-8fd8-465fc5073e0c" />


## Walkthrough

### Step 1: Mine the Brochure for Clues

Download and read the task file carefully. The brochure yields **three clues** — the hotel's identity and the names connected to it. In OSINT, a printed artefact is a set of search terms; write each one down before going online.

<img width="2051" height="1636" alt="step-2" src="https://github.com/user-attachments/assets/30f5f5e4-b7a5-4876-a485-9205df075f93" />


### Step 2: Find the Hotel's Account

Search for the hotel by name on social media: an account exists under the **hotel's name**. That account is the hub — a business's profile links, in public, to the people around it.

<img width="2555" height="1414" alt="step-3" src="https://github.com/user-attachments/assets/2f321601-4bbd-4079-8a34-92b7af2dadd5" />


### Step 3: Pivot Through the Following List to Vera

Open the hotel account's **following** list. Among the followed accounts is **Vera** — a name that also appears in the brochure. That cross-reference is the confirmation that this is the right person, not a coincidence of names.

<img width="2413" height="1147" alt="step-4" src="https://github.com/user-attachments/assets/b2a3678b-529c-480c-830e-ce8ea75ee828" />


### Step 4: Three Base64 Strings on Vera's Profile

Open Vera's account. Posted/displayed on her profile are **three Base64-encoded strings** — individually they decode to fragments, which is the tell that they belong together.

<img width="2525" height="1575" alt="step-5" src="https://github.com/user-attachments/assets/1b02cd0e-e945-4b50-9fe4-d87767ff90c5" />


### Step 5: Join, Decode — Flag

Concatenate the three strings in order and Base64-decode the result (CyberChef's "From Base64" handles it in one step). The plaintext is the **flag** (value in the screenshot, not repeated here).

<img width="1531" height="1400" alt="step-6" src="https://github.com/user-attachments/assets/76640048-e48c-4fa0-a932-298d2fb90c89" />


## Key Takeaways

- Offline artefacts start online investigations: a brochure's names became search terms, and every clue in it was used. Inventory the evidence before touching a browser.
- Following/follower lists are public relationship maps. The pivot from a business to a person cost one click because the connection was displayed, not hidden.
- Cross-referencing is verification: Vera mattered because her name appeared in *both* the brochure and the following list. One match is a lead; two independent matches are identification.
- Split encodings still decode: a secret broken into three Base64 fragments across a profile is reassembled by joining before decoding — and Base64 remains an encoding, never a protection.

## Remediation

For individuals and businesses alike, the lesson is that public social graphs are intelligence: a business account's following list can expose staff and associates, and anything posted in an "encoded" form is effectively published in clear. Keep personal accounts unlinked from business ones where exposure matters, review what profiles reveal in aggregate (each fragment looks harmless alone), and never treat Base64 or any encoding as a way to share secrets.
