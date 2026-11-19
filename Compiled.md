# Compiled

**Platform:** TryHackMe
**Category:** Challenge Room (binary analysis)
**Difficulty:** Easy
**Status:** ✅ Solved (password recovered)

## Background

Compiled is the smallest possible reverse-engineering lesson: you are given a **compiled ELF binary** and asked for the password it checks. The room even warns that the binary **will not execute in the AttackBox** — which is the point. A program that compares your input against a stored password must contain that password, and compilation does not encrypt anything: the comparison value sits in the binary as a plain string, next to the `strcmp` import that gives the game away. `file` identifies the target, `strings` reads its text, and the only trap is a decoy — the most obvious candidate string, the one sitting by the "Password:" prompt, is a format string and does **not** work. The real password is the value the comparison actually uses, in the answer's `*********_****` shape.

## Lab Objective

Download the task binary, identify it, and recover the password it checks from its embedded strings — without running it.

**Provided resources:** the task file (also present in the AttackBox under `/root/Rooms/Compiled/`).

<img width="2589" height="912" alt="step-1" src="https://github.com/user-attachments/assets/da838b88-dd35-48ff-be1f-16162ce02c21" />

<img width="3324" height="940" alt="step-2" src="https://github.com/user-attachments/assets/72580a8c-4639-4c95-ac04-b88af6ec7e99" />


## Walkthrough

### Step 1: Get the Binary and Read the Brief

Download the task file from the room (Task 1). The brief asks a single question — *"What is the password?"* — with the answer format `*********_****`, and notes that the binary will not execute in the AttackBox but the challenge can still be solved. That note is the method in disguise: this is a **static analysis** exercise.

### Step 2: Identify the File

Run `file` on the download (`Compiled-1688545393558.Compiled`):

```bash
file Compiled-1688545393558.Compiled
```

It is an **ELF 64-bit LSB PIE executable, x86-64, dynamically linked, not stripped** — an ordinary Linux program, with its symbols and strings intact.

<img width="2826" height="339" alt="step-3" src="https://github.com/user-attachments/assets/a630df3a-dee2-4fa1-9c34-d329167197f3" />


### Step 3: Read Its Strings

A password checker must store what it checks against, so dump the printable strings:

```bash
strings -n 4 Compiled-1688545393558.Compiled
```

The imports already tell the story — **`scanf`** (reads your input), **`printf`/`fwrite`** (prints the prompts) and **`strcmp`** (compares the two). Among the data strings sit a `Password:` prompt and a password-looking candidate right beside it.

<img width="2653" height="1394" alt="step-4" src="https://github.com/user-attachments/assets/0496851d-ee53-4259-b389-d86243182789" />


### Step 4: Reject the Decoy, Take the Real Password

The obvious candidate — the string by the prompt — **does not work**: it is a `printf` *format string* (it contains a `%s` placeholder), i.e. the text the program prints, not the value it compares. The password that `strcmp` actually tests against is the other string the binary carries, matching the room's `*********_****` format (its value is visible in the screenshot and is not repeated here). Submit it — room complete.

## Key Takeaways

- Compiled is not encrypted. Any secret a binary compares against, prints, or connects with is recoverable from the file itself — `strings` is the first tool, before any disassembler.
- Read the imports to understand a program before reading its data: `scanf` + `strcmp` announced the entire mechanism (prompt, read, compare) in two lines.
- Beware the string that *looks* like the answer. Format strings (`%s`, `%d`) are templates the program fills in — the decoy here sat in the most visible position precisely because prompts are printed, while the compared value is what matters.
- Client-side password checks are not a control: if the checker ships to the user, the password ships with it. Real verification happens server-side, against hashes, never in a binary the attacker can read.

## Remediation

Never embed passwords or secrets in shipped binaries — anything distributed can be read with `strings`, and "compiled" is not a protection layer. Perform credential checks server-side against salted hashes, keep secrets in environment or a secrets manager rather than source code, and treat any client-side check as a UX convenience only. Where a binary must hold configuration, assume it is public and design accordingly.
