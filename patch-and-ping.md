# patch-and-ping

An automation tool that patches, reboots, and confirms your fleet comes back online -- all with a real-time dashboard and summary output.

---

## Why I Built It

Managing a handful of reboot-requiring updates is fine, but across a larger fleet, the effort adds up fast.

Case in point: there was a stretch of time where frequent security kernel updates turned a tolerable event into a recurring disruption. That cycle was eating up hours -- patching, flipping between screen tabs, rebooting boxes, and pinging them to confirm they were back.

So I built `patch-and-ping` to automate the entire process.

---

## Design Overview

`patch-and-ping` connects to a list of target hosts via SSH and orchestrates updates, reboots, and validation checks in parallel. It uses a simple per-host state model and displays everything in a dynamic, auto-refreshing terminal dashboard.

Behind the scenes, it:

1. Runs package updates on each host (using the local `yum` or `dnf`)
2. Captures progress in real-time (e.g. "24/46 packages")
3. Reboots the host if the update is successful
4. Monitors until the host comes back online
5. Groups results by status and generates a summary report

Each server's state is tracked independently through temporary files that record execution details and feed the dashboard. A SIGINT trap ensures that pressing Ctrl+C gives you a clean summary rather than leaving things in an unknown state.

---

## Sample Output (Test Mode)

## Test Mode Support

The script includes a test mode that simulates a live run without touching any real servers. For example:

```bash
# echo "Success" > /tmp/status_alpha.txt
# echo "Update failed: No YUM/DNF found" > /tmp/status_bravo.txt
# echo "Failed to come back online in 420 sec" > /tmp/status_charlie.txt
# echo "Excluded" > /tmp/status_delta.txt
# echo "Unknown error during update" > /tmp/status_echo.txt
# ./patch-and-ping.sh --test-summary
```

This would generate a live dashboard like this:

```bash
SERVER               STATUS                                      VIEW LOG    
alpha                Success                                     /tmp/status_alpha.log
bravo                Update failed: No YUM/DNF found             /tmp/status_bravo.log
charlie              Failed to come back online in 420 sec       /tmp/status_charlie.log
delta                Excluded                                    /tmp/status_delta.log
echo                 Unknown error during update                 /tmp/status_echo.log
```

When all updates complete (or if you interrupt with Ctrl+C), it shows a summary report:

```bash
--- Update and Reboot Summary ---
Date: Mon May 18 14:30:22 PST 2025

SUCCESSFUL
alpha: Update & Reboot Successful

EXCLUDED 
delta: Excluded from processing

FAILED
bravo: Update failed: No YUM/DNF found
charlie: Failed to come back online in 420 sec
echo: Unknown error during update

Report saved to: /home/username/patch-and-ping_051825.txt
```

❗ Want to see what the dashboard looks like in action? [`Check out a visual example.`](./patch-and-ping-dashboard.md)

---

## Usage

I typically feed the script with a list of servers generated from an internal scope-aware tool, but any standard way of supplying the host list is acceptable. Beyond that, the script also accepts optional flags for job control, exclusions, and test mode (most of which can also be configured directly in the script):

```
Usage: patch-and-ping.sh [options] { <server1> [<server2> ...] | $(command producing server list) }

Options:
  -j, --jobs            Maximum parallel jobs (default: 20)
  -t, --timeout         Server recovery timeout in seconds (default: 420)
  -x, --exclude         Comma-separated list of servers to exclude
  -T, --test-summary    Show summary output using simulated /tmp/status_<host>.txt files
  -h, --help            Show this help message

Examples:
  Supply servers directly as command-line arguments:
    ./patch-and-ping.sh foo bar

  Supply a list of servers from another command (one per line):
    ./patch-and-ping.sh $(~/src/ficonfig/classmates rocky)
```

---

## Dashboard and Job Coordination

One of the most useful aspects of this tool is its live dashboard -- which refreshes in real time while servers are being updated and rebooted in parallel. This behavior is coordinated using background jobs, `xargs`, and traps, like so:

```bash
# Launch the dashboard logic (not actual terminal view) in the background
display_dashboard &
dashboard_pid=$!

# Disown to suppress termination notifications
disown "$dashboard_pid"

# Launch update jobs via xargs in a subshell
(
    printf '%s\n' "$@" | xargs -P "$MAX_PARALLEL_JOBS" -I {} bash -c "$CMD" -- {}
) &
xargs_pid=$!
```

---

## Notable Features

- **Parallel Processing**  
  Updates multiple servers simultaneously (with a configurable concurrency limit)

- **Live Dashboard**  
  See real-time status, with automatic refreshing, across all machines

- **Smart Monitoring**  
  Tracks when servers go down and come back up with timeout enforcement

- **Categorized Reporting**  
  Results alphabetized and grouped by status importance (Success, Excluded, Failed)

- **Exclusion Lists**  
  Built-in and command-line options for skipping specific hosts

- **Safe Interruption**  
  Ctrl+C produces a clean summary rather than abandoning the process

- **Test Mode Support**  
  Simulates a full run with fake hosts and randomized output for various testing

- **Color Coded Output Support**
  Displays statuses in green, yellow, or red based on severity for easy scanning

---

## Reflection

I'm proud of my execution here. I resisted the urge to bring in heavier tools like `dialog` or `whiptail`, and instead built something that fits the Unix mindset: *do one thing well*.

And now, when this task comes up, I’m actually happy to do it -- which happens with, on average, a 75% time savings. Plus, the newly automated nature gives me space to multi-task a bit in the background. A win for me; a win for my employer.
