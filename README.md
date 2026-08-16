[README.md](https://github.com/user-attachments/files/31124814/README.md)
# WI5 – Calculate Client Security Hash

A UiPath RPA solution that automates the **Calculate Client Security Hash** process against the ACME System 1 training web application (`acme-test.uipath.com`), as specified in the Process Design Document (PDD).

## Process Overview

For every Work Item of type **WI5 – Calculate Client Security Hash**, the robot:

1. Logs into ACME System 1
2. Retrieves the full list of Work Items (paginated)
3. Filters for items where `Type = "WI5"`
4. For each WI5 item:
   - Opens the item's details page
   - Extracts **Client ID**, **Client Name**, and **Client Country**
   - Builds the input string `ClientID-ClientName-ClientCountry`
   - Computes the **SHA1 security hash** of that string
   - Opens the item's Update page
   - Adds the hash as a comment
   - Sets the status to **Completed**

## Architecture

This solution uses a **single-workflow design** (`Main.xaml` → `Login.xaml`) rather than a Dispatcher/Performer split, since the process is not queue-based and does not require Orchestrator transaction distribution.

### Key design decisions

- **Direct URL navigation** is used wherever possible instead of clicking through the UI, since ACME System 1's pages follow predictable, stable URL patterns:
  - Work Items list (paginated): `https://acme-test.uipath.com/work-items?page=N`
  - Work Item details: `https://acme-test.uipath.com/work-items/{WIID}`
  - Update Work Item: `https://acme-test.uipath.com/work-items/update/{WIID}`

  This avoids fragile, icon-based selectors and makes the automation far more resilient to UI changes.

- **SHA1 hash is computed in-process (C#)** rather than by automating the external sha1-online.com website. This produces an identical, correct result while removing dependency on a third-party site's availability and UI structure.

- **Credentials** are retrieved securely via `Get Secure Credential`, pulling from **Windows Credential Manager** (`Target = "ACME_System1"`) — never hardcoded.

## Prerequisites

1. **UiPath Studio** (this project uses C# as the expression language)
2. **Windows Credential Manager** entry:
   - Generic credential named `ACME_System1`
   - Username/Password = your ACME System 1 test account
3. An ACME System 1 test account, registered at `https://acme-test.uipath.com/register`
4. (Optional, for Orchestrator execution) A UiPath Orchestrator tenant with:
   - A folder for this project
   - Your machine connected via UiPath Assistant

## Setup

1. Clone/download this repository.
2. Open `project.uiproj` in UiPath Studio.
3. Add the `ACME_System1` credential to Windows Credential Manager on the machine that will run the robot.
4. If running via Orchestrator: publish the project (`Publish` in Studio), create a Process in your Orchestrator folder from the published package, and run it as an **Attended** job via UiPath Assistant (no unattended machine credentials required for this setup).
5. Before each run, ensure test data is reset in ACME System 1: **User options → Reset Test Data**.

## Project Structure

```
WI5_Calculate_Client_Security_Hash/
├── Main.xaml                  # Entry point — invokes Login.xaml
├── Workflows/
│   └── Login.xaml             # Core process: login, pagination, WI5 filter,
│                               # per-item hash computation and update
├── Data/                      # (reserved for future config, unused currently)
├── project.json
└── project.uiproj
```

## Known Issues & Lessons Learned

- **Never set `ElementVisibilityArgument` or `WaitForReadyArgument` to `"Interactive"`** on UI Automation activities against this site. This setting caused repeated indefinite hangs with no error surfaced during development. All activities in this project use `"None"` instead.
- **Do not enable "Verify"/retry options on Type Into activities** — this can cause a type-then-clear loop on some fields.
- Multi-page table extraction via the built-in "Next Link" pagination feature proved unreliable on this site; this project instead loops through page URLs directly (`?page=1`, `?page=2`, ...) and merges each page's extracted table into a master `DataTable`.

## PDD Reference

This implementation follows the Process Design Document: **"Calculate Client Security Hash"** (ACME Systems Inc., Finance and Accounting department), including the specified exception handling for incorrect credentials and missing WI5 tasks.
