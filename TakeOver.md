# TakeOver

**Platform:** TryHackMe
**Category:** Challenge Room
**Difficulty:** Easy
**Status:** ✅ Solved (flag captured)

## Background

TakeOver is a pure **subdomain enumeration** room, built around a truth about how companies actually get breached: the main website is hardened and watched, while forgotten subdomains are neither. The target is `futurevera.thm`, and its front door yields nothing — the interesting surface is sideways. Virtual-host fuzzing finds two subdomains, `blog` and `support`. The chain finishes with the room's signature trick: the **TLS certificate** served by the support subdomain lists a *Subject Alternative Name* — a "secret" helpdesk subdomain with a randomised name (`secrethelpdesk934752.support.futurevera.thm`) that no wordlist would ever guess. Certificates are public by design, so the secret was published the day the cert was issued. Add the name to `/etc/hosts`, browse to it, and the flag is simply there.

## Lab Objective

Enumerate the target's subdomains by virtual-host fuzzing, then mine the support subdomain's TLS certificate for the hidden helpdesk subdomain and capture the flag from it.

**Provided resources:** the target machine only; no credentials are given.

<img width="3042" height="936" alt="step-1" src="https://github.com/user-attachments/assets/23a4f4a2-5ac5-43c4-b56e-bd1648c2ee26" />


<img width="1985" height="821" alt="step-2" src="https://github.com/user-attachments/assets/0d494160-1225-43d5-90f9-ffb13a0bb4fc" />


<img width="2249" height="1009" alt="step-3" src="https://github.com/user-attachments/assets/886211b8-3e80-4350-9451-09deae2d6a5f" />


## Walkthrough

### Step 1: Set Up and Look at the Main Site

Map `futurevera.thm` to the target IP in `/etc/hosts` and browse the main site. Check what it exposes — including the `/assets` directory, which the server happily lists. It contains only a favicon and an images folder: **nothing of value**. The front door is a dead end, which is the room's way of saying the attack surface is elsewhere.

<img width="2841" height="1141" alt="step-4" src="https://github.com/user-attachments/assets/743fac8d-beba-4865-aec1-e9064c045253" />


<img width="2025" height="481" alt="step-5" src="https://github.com/user-attachments/assets/83760738-832d-45e5-8c4f-d7c7cb98dddd" />


<img width="1637" height="766" alt="step-6" src="https://github.com/user-attachments/assets/3c9dbb46-8595-4820-b58f-6ef17edf57e9" />


### Step 2: Fuzz Virtual Hosts — `blog` and `support`

When one IP hosts many sites, the rest are found by fuzzing the `Host` header against a subdomain wordlist (ffuf's vhost mode, or Gobuster's `vhost` mode):

```bash
ffuf -u https://futurevera.thm -H "Host: FUZZ.futurevera.thm" -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt -fs <default-size>
```

Two subdomains answer with distinct responses:

- `blog.futurevera.thm`
- `support.futurevera.thm`

Add both to `/etc/hosts` so the browser can reach them, and visit them.

<img width="2409" height="642" alt="step-7" src="https://github.com/user-attachments/assets/8bb59958-4bdd-46a6-a5fc-f6142ed8a794" />


### Step 3: Read the Support Site's Certificate — the SAN Gives Away a Third Subdomain

Open the support subdomain over HTTPS and inspect its **TLS certificate** in the browser (padlock → certificate details). The certificate's **Subject Alt Names** field lists a DNS name that fuzzing never found:

```text
secrethelpdesk934752.support.futurevera.thm
```

A subdomain of the support subdomain, with a long randomised label — deliberately unguessable by wordlist. It did not need guessing: a certificate must name every host it covers, and certificates are served to anyone who connects. "Secret" names protected only by obscurity are published in the very credential that secures them.

<img width="2535" height="1555" alt="step-8" src="https://github.com/user-attachments/assets/8c975b8c-ec58-40bc-ba7b-ecd7b6844a56" />


### Step 4: Visit the Hidden Helpdesk — Flag

Add the discovered name to `/etc/hosts` alongside the others, then browse to it over plain HTTP. The hidden helpdesk page serves the room's **flag** (value in the screenshot, not repeated here).

<img width="2126" height="1071" alt="step-9" src="https://github.com/user-attachments/assets/18ce0c96-986a-4249-aa85-1043602c1aad" />


## Key Takeaways

- Subdomains are the soft perimeter: the main site showed nothing, while two fuzzed vhosts and one certificate-leaked vhost carried the whole room. Enumeration has to cover *names*, not just ports and paths.
- Vhost fuzzing works because the server answers differently for names it knows — filter by the default response size (`-fs`) so real hosts stand out from the catch-all.
- TLS certificates are public documents. Subject Alternative Names enumerate exactly the hosts an organisation wanted to keep quiet, and certificate-transparency logs make the same data searchable at scale — never rely on an unguessable hostname as a security control.
- Every discovered name goes in `/etc/hosts` (in a lab) and into scope notes (in real work) immediately; half of enumeration is bookkeeping.

## Remediation

Treat subdomain inventory as a security control: maintain an authoritative list, decommission forgotten hosts, and put authentication in front of internal tools like helpdesks instead of relying on obscure names — assume every hostname in a certificate is public knowledge, because it is. Where a service must stay private, keep it off public DNS and certificates entirely (internal CA, split-horizon DNS, VPN-only access). Finally, monitor certificate-transparency logs for your own domains; the same feed that leaks names to attackers lets defenders spot shadow IT first.
