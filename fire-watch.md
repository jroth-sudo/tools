# fire-watch

A comprehensive temperature monitoring and safety system -- covering both UPS internals and overall room heat.

## What it does

This solution provided an environmental safety layer across ~25 UPS units and the machine room itself. It monitored internal UPS temperature readings, tracked ambient temperature via a serial sensor, and tied everything into Nagios and Cacti. If temperatures exceeded set thresholds, alerts were triggered and (in certain scenarios) automated shutdowns were initiated using `apcupsd`.

Key capabilities:
- Centrally queried and monitored UPS temperatures
- Monitored ambient room temperature via wall-mounted sensor
- Alerted via Nagios if values exceeded acceptable levels
- Triggered clean shutdowns during critical events using apcupsd hooks
- Logged values for historical graphing in Cacti

## How it works

### UPS Temperature Monitoring

I purchased APC units with onboard temperature support (`ITEMP`). A shell script queried these values using `apcaccess`, converted the results to Fahrenheit, and flagged anything over 101°F:

```bash
upstemp=`$apcaccess status $ups | awk '/ITEMP/ { print $3 }'`
upsftemp=`echo "$upstemp * 1.8 + 32" | bc`
```

Nagios polled this information every 30 minutes.

### Ambient Room Monitoring

A separate wall-mounted TempTrax sensor provided room-level heat tracking. A wrapper script called a third-party plugin over serial. The same script wrote values to a spool file so Cacti could graph trends over time:

```perl
my $output=`sudo /usr/lib64/nagios/plugins/check_temptraxf -d /dev/ttyS0 -p 1 -w $warn -c $crit`;
...
open (F, ">$tmp");
print F "$temp
";
rename($tmp, $file);
```

This helped identify faulty UPS units, failing AC, or spikes during power/load changes.

### Automated Shutdowns

Hosts were configured as apcupsd clients, polling a central apcupsd server over the network. When serious UPS conditions occurred -- such as low battery runtime -- the client’s handlers took over, triggering custom notifications, logging, and clean shutdowns via `apccontrol`.

The 101°F threshold was used for early warning -- prompting alerts and investigation, but not direct shutdowns. That said, the system retained the ability to shut down automatically in rare thermal scenarios where internal temperatures reached levels that clearly indicated risk of damage or fire.

### Graphing and Visibility

Both room and UPS temperatures were plotted via Cacti. This was useful not just for real-time monitoring but for forensics and trend analysis (such as identifying faulty units or failing batteries).

## Reflections

This solution came from necessity. As a self-ran machine room, we had no managed power or HVAC infrastructure, so I built what was needed.

Furthermore, because our facility used a standard water-based fire suppression system, prevention wasn’t just about avoiding heat damage -- it was about avoiding a chain reaction that could destroy the entire room.

The stack was built entirely with open source tools (Nagios, apcupsd, Cacti) and a small number of vendor components. I've since retired this system after moving to a proper datacenter but at the time, it was essential, and it worked.
