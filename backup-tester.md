# backup-tester

A distributed data validation system to prove backup integrity across platforms.

## Why I Built It

Backups are only useful if they can be restored. Many systems check whether backups exist -- fewer verify that those backups can actually be trusted. I built this solution to provide real, continuous assurance that the backup archives were restorable across different operating systems, architectures, and formats. This became even more important once backups began being replicated offsite.

## Design Overview

The system consists of two scripts: a central orchestrator and a host-side validator. The orchestrator runs from a backup server and triggers restore tests on each platform via SSH. Each client fetches its most recent archive and attempts to list its contents. Results are pushed to Nagios using a wrapper and alerts are raised when something goes wrong.

## How It Works

### Server Orchestration

The main script runs nightly via cron. It defines multiple host groups based on backup file format and platform architecture:

```bash
fsdump_hosts=(alpha bravo charlie)
dump_hosts=(delta echo foxtrot)
```

Each host is contacted via SSH, where it runs its local restore-check logic. These host groups are designed to align exactly with the expected restore tooling (which is based on whether the backup is a full volume or partial filesystem backup), OS, and 32/64-bit compatibility -- ensuring each backup is tested in its native environment.

### Client Validation

Each client:
- Determines which archive path to test based on hostname
- Connects to the backup server to pull the latest level 0 (or level 1 fallback) archive
- Pipes the archive into the appropriate restore command in list mode
- Checks the exit status
- Failures are reported to Nagios with context (platform, path, error)

### Nagios Integration

A wrapper script (`send2nagios`) captures script output and pushes status to Nagios via NSCA. By default:
- An empty (successful) run results in an OK status
- Any output results in a CRITICAL alert with logs attached

This ensures alerts are surfaced only when something fails (keeping noise low but visibility high).

## Notable Features

- **Cross-Platform Validation**  
  Covers combinations of OS and architecture, ensuring that each restore is tested on its native platform using the correct tool.

- **Format Awareness**  
  Differentiates between backup formats, invoking the appropriate restore logic based on archive type.

- **Smart Archive Selection**  
  Automatically finds the most recent backup, ensuring that the test always runs against current, relevant data.

- **Nagios Integration**  
  Pushes results to Nagios using a wrapper, with alerts triggered only on failure.

## Reflections

This solution is a core piece of Disaster Recovery. We already had backups but this took our strategy end-to-end. It caught several issues over time, from software bugs to unexpected failures, all before they became real incidents.
