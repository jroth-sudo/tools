# service-sentinel

An intelligent high-availability routing system that automatically detects service failures and redirects traffic accordingly.

---

## Why I Built It

Our corporate web, DNS, and mail services needed to remain available -- even during maintenance windows or unexpected outages. Traditional load balancers would’ve added more complexity (and cost) than the situation really called for, but doing nothing meant risking downtime for critical public-facing infrastructure.

As such, my solution provided a way to reroute traffic dynamically without adding anything except a few scripts on top of the existing firewall config. It's simple yet reliable, and has successfully handled failover for years.

---

## Design Overview

`service-sentinel` runs on a perimeter firewall and works in tandem with iptables. It:

1. Performs regular health checks across key services (HTTPS, SMTP, DNS)
2. Automatically rewrites prerouting rules based on those checks
3. Sends e-mail notifications when servers go down or recover
4. Allows maintenance or reboots without causing user-visible disruptions

When both servers are healthy, traffic routes in a round-robin fashion. If one fails any of the checks, it’s immediately pulled from the active routing table until it passes again. The result is a self-correcting infrastructure layer that requires almost no manual intervention.

---

## How It Works

Every minute, the daemon-like script:

- Pings each server to confirm basic network connectivity  
- Uses `wget` to verify HTTPS is live (optionally checking a debug endpoint)  
- Runs a custom probe script to confirm SMTP responsiveness  
- Performs an `nslookup` to validate DNS resolution for the expected domain

If any of these checks fail:

- The server is marked as "down"  
- It’s removed from the NAT/prerouting rules  
- A detailed notification is sent to the SA team, describing the type of outage

When all checks pass again:

- The server is marked as "up"
- It’s reintroduced to the routing table
- A recovery notice is sent

The routing logic itself is handled via a separate iptables management script that rebuilds NAT rules based on the current status of the monitored hosts.

---

## Notable Features

- **Multi-service health checks**  
  Goes beyond ping to confirm actual service-level availability

- **Zero-downtime maintenance**  
  Servers can be taken offline cleanly without user impact

- **Self-healing behavior**  
  Automatically adapts to outages and recovers without intervention

- **No external dependencies**  
  Built entirely with standard CLI tools and scripts

- **Reusable pattern**  
  Though originally built for a specific use case, the same approach can be (and has been) used for virtually any service

- **Scalable design**  
  While this write-up focuses on a two-server setup, the underlying idea easily supports larger server pools with minimal modification

---

## Reflection

Two things I appreciate most about this tool: it catches issues that traditional monitoring can’t, and it has prevented real outages before users even noticed something was wrong.

While it’s not a traditional load balancer, it packs a lot of value without the overhead.

One design tradeoff worth noting: the tool currently pulls a host entirely out of rotation if any single service fails -- even if others remain healthy. In the case it was written for, that’s a safe and acceptable behavior since the servers are fully mirrored and lightly loaded. That said, if the environment ever called for more granular handling (e.g. per-service routing), the pattern could be adapted to support it.
