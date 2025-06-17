# apply-config

`apply-config` is a lightweight Bash script I wrote to quickly and consistently set up servers and manage configurations. It supports both class-based inheritance and host-specific logic, making it easy to define modular or fully tailored setups as needed.

It was part of an internal system called `ficonfig`, built around a Git repository with minimal external dependencies. At the time, tools like Ansible weren’t yet widely adopted. Building something custom gave us direct control and also aligned with our team's engineering culture, which encouraged writing internal tools when it was practical and worthwhile.

“Lightweight” here means there’s no agent and no required tooling beyond Bash, SSH, and rsync -- just a single script running logic over a versioned config tree. This made it especially useful for freshly installed systems with hardened or minimal OS baselines, where extra tooling wasn’t yet available or permitted.

---

## Design Overview

- Reads class and host logic from a Git-backed directory structure
- Syncs configuration files and scripts to target hosts using `rsync` over SSH
- Interprets config class membership via `/etc/config-classes`
- Applies `config-scripts/` after syncing, in lexicographic order (to enforce execution priority)
- Supports dry run and rsync-only modes
- Allows host simulation to preview config application logic (useful for offline hosts or testing)

---

## Layout Overview

```
ficonfig/
├── apply-config
├── classes/
│   ├── linux/
│   │   ├── etc/
│   │   └── config-scripts/
│   │       └── prep-time
│   ├── rocky-8/
│   └── webserver/
├── hosts/
│   ├── foo/
│   │   └── etc/
|   |       ├── config-classes
│   │       ├── custom.conf
│   │       └── config-scripts/
│   │           └── 001-restart-httpd
```

---

## Class-Based Inheritance Model

Each host defines its config classes in `/etc/config-classes`, in order from general to specific. For example:

```
base linux rocky-8 webserver
```

This order determines:
- Which files/scripts apply
- Which ones override earlier ones

---

## Typical Execution Logic

For each host, `apply-config`:

1. Reads its `/etc/config-classes`
2. Syncs all matching files (class + host-specific)
3. Makes `config-scripts/` executable and runs them in order (unless using `--rsync-only`)

Example usage:

```bash
apply-config foo
```

Common flags:

| Flag            | What it does                             |
|-----------------|------------------------------------------|
| `-v`            | Verbose output                           |
| `-n`            | Dry run                                  |
| `-c`            | Apply against entire class               |
| `-h`            | Simulate for a different host            |
| `--rsynconly`   | Only sync files, don't run scripts       |
| `--remote`      | Use different Git repo                   |

---

## Notable Features

- **config-scripts model**  
  Shell scripts stored per class/host. They're run alphabetically post-sync and can perform any programmable logic.

- **Multi-platform support**  
  The class system handles different OSes/distros (e.g. `linux`, `rocky-8`, `solaris`), so configs and scripts apply only where relevant.

- **Per-version package targeting**  
  Files in `/etc/packages/` allow finer-grained logic based on distro version (e.g. using `ntp` for CentOS 7 and `chrony` for Rocky 8).

- **Package removal support**  
  Packages prefixed with `!` are not installed -- and are uninstalled if present. This helps eliminate conflicts, cruft, and potential security issues.

- **Dry-run mode**  
  Provides a safe way to preview changes before touching production systems.

---

## Sample config-script: `prep-smartd`

```bash
#!/bin/bash
# ficonfig/classes/linux/etc/config-scripts/prep-smartd
#
# Enables and starts the smartd service, then restarts it if the config file was
# modified after the last service start time. Uses helper scripts for systemd/sysv
# compatibility and timestamp comparison.

/usr/local/sbin/control-service smartd enable

PID=`pidof -s /usr/sbin/smartd`

if [ -f /etc/smartd.conf ]; then
    CONF="/etc/smartd.conf"
elif [ -f /etc/smartmontools/smartd.conf ]; then
    CONF="/etc/smartmontools/smartd.conf"
fi

/usr/local/sbin/control-service smartd start

if /usr/local/bin/file-newer-than-pid $CONF $PID; then
    /usr/local/sbin/control-service smartd restart
fi
```

---

### Supporting helper: `control-service`

```bash
#!/bin/bash
# ficonfig/classes/linux/usr/local/sbin/control-service
#
# Cross-platform wrapper to run either `systemctl` or `service` based on what the
# host supports (sysv vs systemd).

servicename=$1
operation=$2

if [ -x /usr/bin/systemctl ]; then
  /usr/bin/systemctl $operation $servicename.service
else
  /sbin/service $servicename $operation
fi
```

---

### Supporting helper: `file-newer-than-pid`

```bash
#!/bin/bash
# ficonfig/classes/linux/usr/local/bin/file-newer-than-pid
#
# Check if a file was modified more recently than a given PID's process start time.

filename="$1"
pid="$2"

if [ ! -f "$filename" ] || [ ! -d "/proc/$pid" ]; then
    echo "Usage: $0 <filename> <pid>" >&2
    exit 1
fi

# Get system uptime in seconds
uptime=$(cut -d. -f1 /proc/uptime)

# Get process start time in jiffies (21st field of /proc/$pid/stat)
start_jiffies=$(awk '{print $22}' /proc/$pid/stat)

# Get number of jiffies per second
jps=$(getconf CLK_TCK)

# Convert to seconds since boot
start_seconds=$((start_jiffies / jps))
process_start_time=$((uptime - start_seconds))

# Convert to epoch time
process_start_epoch=$(date +%s)
process_start_epoch=$((process_start_epoch - process_start_time))

# Get file modification time
file_mtime=$(stat -c %Y "$filename")

# Compare times
if [ "$file_mtime" -gt "$process_start_epoch" ]; then
    exit 0  # true: file is newer
else
    exit 1  # false: file is older
fi
```

---

## Reflection

Though I’ve come to appreciate tools like Ansible -- and have been gradually migrating toward it -- this system has worked reliably for a long time and served as a foundational piece of infrastructure. It reflects my pragmatic approach: building just enough to solve the problem clearly and sustainably -- and, for me, it was a genuinely satisfying technical challenge.
