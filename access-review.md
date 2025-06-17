# access-review

A single-command access audit tool that gathers account data across devices and cloud services.

---

## Why I Built It

Quarterly access reviews are a required part of SOC 2 compliance -- but manual reviews across a mix of systems, identity sources, and cloud accounts can get messy fast. So I wrote this tool to standardize the process, increase visibility, and support reliable, sign-off-ready reviews.

While originally built to support SOC 2, I further decided to extend the scope to include **all infrastructure accounts**, not just those narrowly defined under compliance. The result is a single command that pulls data from all relevant systems and providers, producing a clean, actionable report.

---

## Design Overview

My shell-based solution connects via SSH or REST API, depending on the nature of the system (e.g. local vs. cloud), then gathers account information from all sources. The result is a consistent, human-readable summary.

The summary includes:

### System-level accounts:
- Pulls centralized accounts via LDAP
- Pulls local accounts from `/etc/passwd`
- Excludes known system/service accounts (e.g., `nobody`, `daemon`, etc.)
- Excludes shell-less accounts (`/usr/bin/false`, `/usr/sbin/nologin`) since these lack actual access
- Flags locked accounts that haven’t been removed, for manual follow-up

### Cloud service accounts:
- **Google Workspace**: Lists user accounts and 2-step verification status using the Admin SDK
- **AWS**: Lists IAM user accounts via API
- Other cloud services of interest are included when defined in the Asset Management system (via API)

### Gap detection:
- Any known cloud provider not covered by automatic review is explicitly listed in the output under a *"Manual Review Required"* section
- This ensures full visibility even when API integration isn’t available or supported

---

## Usage Context

The script is run manually from a trusted user account (not root), using `ssh-agent` to connect to each relevant host. Output is placed into a shared document and sent to leadership (e.g. the COO) for review and sign-off.

Each report kicks off a brief internal review cycle where lingering accounts or policy exceptions are discussed and resolved. Follow-up actions -- as well as final sign-off -- are tracked in an internal ticket for audit traceability.

---

## Notable Features

- **Multi-source access visibility**  
  Gathers account data from both local systems and LDAP in a single sweep

- **RBAC-aware SSH visibility**  
  Parses `sshd_config` to capture AllowUsers/AllowGroups directives, ensuring login rights reflect actual access policy

- **Cloud account integration**  
  Pulls account information from cloud providers using their respective APIs

- **External access auditing**  
  Audits configuration for systems that control SSH exposure from the outside, identifying which users are authorized to open external access

- **Inventory-based scoping**
  Directly leverages asset management data to define which systems and providers should be included or flagged for manual review

- **System/service account filtering**  
  Automatically filters out noise from known system accounts and login-disabled shells

- **Locked account detection**  
  Flags accounts that are locked but still present, for follow-up

- **Management-friendly output**  
  Outputs column-aligned summaries designed for easy review and sign-off by less technical stakeholders

---

## Reflection

SOC 2 was a great project; it gave me a lot of new angles to think about. But it also introduced a whole new set of recurring responsibilities. Making that entire chain less painful through automation — from data gathering to upper management sign-off — felt like a real win.

There are still a few features I’d like to build out, but even as-is, this script has caught things that would’ve otherwise slipped through the cracks. For a single shell script, it’s pulled a surprising amount of organizational weight.
