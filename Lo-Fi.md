# Lo-Fi

**Platform:** TryHackMe
**Category:** Challenge Room
**Difficulty:** Easy
**Status:** ✅ Solved (flag captured)

## Background

Lo-Fi is a one-move **Local File Inclusion (LFI)** room dressed as a lo-fi music site. The site's pages are not separate files in the navigation — they are loaded through a **`page` parameter** in the URL, which the server resolves to a file on disk and includes. Any parameter that names a file is an invitation to name a *different* file: replace the page name with a **path traversal** sequence (`../` repeated) that climbs out of the web directory, aim it at the flag file, and the server includes and displays it. The room isolates the single most-tested LFI habit: when you see `?page=something`, the traversal payload is the next thing to try.

## Lab Objective

Find the file-loading parameter on the Lo-Fi site and abuse it with a path-traversal payload to read the flag file outside the web root.

**Provided resources:** the target machine only; no credentials are given.

<img width="3338" height="909" alt="step-1" src="https://github.com/user-attachments/assets/8fdbb38d-9386-4fa9-81bc-966b1a1450dc" />


<img width="3207" height="1433" alt="step-2" src="https://github.com/user-attachments/assets/c92c51ca-1b16-4209-bc8e-e748c79cd32e" />


## Walkthrough

### Step 1: Browse the Site

Open the target URL. The site is a small lo-fi music page with a few sections/tracks to click through.

<img width="2810" height="1442" alt="step-3" src="https://github.com/user-attachments/assets/09322e4e-92db-4fb1-a987-d13064ac78dd" />


### Step 2: Spot the File Parameter

Click through to a section — the **Relax** page. Look at the address bar: the page is loaded through a **parameter in the URL** whose value names the page being shown. That shape — a parameter whose value is a page or file name — is the classic LFI candidate, so test it.

<img width="2473" height="1508" alt="step-4" src="https://github.com/user-attachments/assets/f5f1fd54-0387-444c-93d9-414690299e9b" />


### Step 3: Traverse to the Flag File

Replace the parameter's value with a path-traversal payload: a chain of `../` segments that walks up out of the directory the application loads pages from, ending at the flag file's location:

```text
?page=../../../../flag.txt
```

(The exact depth is found by trying — too few `../` stays inside the site folder, extra ones are harmless once the filesystem root is reached.)

<img width="2401" height="1152" alt="step-5" src="https://github.com/user-attachments/assets/4282a9ba-1613-4e91-af23-3776f82cfe84" />


The server resolves the path, includes the file, and prints its contents in the page: the **flag** (value in the screenshot, not repeated here).

<img width="1834" height="768" alt="step-6" src="https://github.com/user-attachments/assets/7762a6c1-0854-4932-88f2-c6694be25895" />


## Key Takeaways

- A parameter that names a page or file (`page=`, `file=`, `template=`, `lang=`...) is an LFI candidate by default — the first test is always a traversal sequence to a known file such as `/etc/passwd`, and here, the flag.
- Traversal depth is trial and error: overshooting to the filesystem root and coming back down the absolute path is the reliable pattern.
- Inclusion is worse than mere reading when code files can be targeted — the same primitive that reads a text flag here becomes log poisoning and code execution elsewhere. Treat any user-controlled include path as critical.
- The fix is never a blacklist of `../` strings; it is to stop letting user input name files at all.

## Remediation

Map parameter values to an allow-list of known pages server-side (an ID-to-file table) instead of using input as a path. Where file names must be accepted, canonicalise the resolved path and verify it stays inside the intended directory, run the application with a user that cannot read sensitive files, and keep flags, configs and credentials outside the web tree entirely.
