# external-ssh-check

Monitors for new external SSH connections by parsing centralized logs, then sends a confirmation request to both the user who connected and the Systems Administration team.

---

## Why I Built It

This script was created to provide real-time awareness of connections to our internal network -- even though the systems are protected by other non-exposing tools, behind a perimeter firewall, etc. The goal wasn’t just to detect logins, but to bring them into human visibility, as part of a broader accountability and response policy.

---

## Design Overview

External SSH access is limited to a small set of hosts. The script watches system logs across those hosts from a centralized rsyslog server, looking for new external IPs. When a new connection is detected, it sends an email to both the Systems Administration team and the user involved -- with clear call-to-action instructions and forensic context.

For example:

```
To: sa@foo.com
Subject: Unidentified external SSH logins
Cc: user@foo.com

May 9 12:34:56 hostname username from 1.2.3.4 (external.network.com)

The addresses above, which an SSH connection was established from,
appear to be new addresses. Please have a look at the connection
related to you then let SA know whether you recognize it or not.

Information below this point is meant for SA use only.

Previous three May connections from username, if any and unique: 
username from 2.3.4.5 (host5.network.com)
username from 6.7.8.9 (6-7-8-9.fiber.dynamic.sonic.net)
username from 10.11.12.13 (ec2-10-11-12-13.compute-1.amazonaws.com)
Previous three Apr connections from jkf, if any and unique: 
username from 6.7.8.9 (6-7-8-9.fiber.dynamic.sonic.net)
username from 208.54.5.171 (mab0536d0.tmodns.net)
```

The message includes hostname resolution for friendliness, connection history from the past two months, and a monthly reset to avoid stale data. It runs from cron every minute and supports a lightweight accountability process, with users expected to reply to confirm (or deny) the connection.

---

## Notable Features

- **Integrates with security policy**  
  Alerts are actionable by design -- they request a reply. This creates a clear handoff from detection to confirmation or escalation.

- **Self-pruning memory model**  
  IP tracking resets each month, balancing freshness and signal-to-noise.

- **DNS resolution of source IPs**  
  Hostnames are shown alongside IPs to improve readability and location recognition.

- **Runs every minute via cron**  
  It’s fast, simple, and doesn’t miss a beat.

---

## Reflection

This script became one of the most valuable tools we rely on. It's simple but gives immediate and important security-related visibility, shifting access from being a silent background activity to something people are aware of and accountable for.

It also reflects something I care about -- that security isn’t just about control, it’s about awareness and participation.
