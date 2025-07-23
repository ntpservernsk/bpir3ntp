**Setting Up a Public Stratum 2 NTP Server on OpenWrt (with Banana Pi BPI-R3)**

[![CC BY-SA 4.0][cc-by-sa-shield]][cc-by-sa]

[cc-by-sa]: https://creativecommons.org/licenses/by-sa/4.0/
[cc-by-sa-shield]: https://img.shields.io/badge/License-CC_BY--SA_4.0-lightgrey.svg

## Table of Contents
[Preface](#preface)

[Configuration Guide](#configuration-guide)

[Monitoring and Management](#monitoring-sources--dynamic-management)

[DHCP Advertisement](#dhcp-advertisement-lan-clients)

[Statistics Collection & Visualization](#statistics-collection--visualization)

## Preface

I successfully configured a stable public Stratum 2 NTP server on a Banana Pi BPI-R3 router, integrating it into the [pool.ntp.org](https://ntppool.org) with maximum load provided by the pool. This was achieved without degrading the router's performance in my home network.
The router runs vanilla OpenWrt 24.10.2 firmware without third-party packages. All instructions apply to this version.

[BPI-R3 official page](https://docs.banana-pi.org/en/BPI-R3/BananaPi_BPI-R3)

The NTP server software used is `chrony`. I was unable to get the standard `ntpd` package working reliably; it consistently lost synchronization with sources, whereas `chrony` operates very stably. Other service options were not tested.

The issue likely stems from the very low precision of the router's clock oscillator. Unfortunately, this oscillator (as indicated by several reports on the official forum) exhibits a significant stable frequency offset for many (if not all) users. For instance, my unit has a consistent offset of approximately 1073 ppm, resulting in an error accumulation of about 1.5 minutes per day. Fortunately, the oscillator's frequency drift (instability) is very low (less than 0.1 ppm in my case), allowing chrony to successfully compensate for this stable offset down to zero. Consequently, the server's operation remains unaffected.

CPU load averages around 10% (on one of four cores). Memory usage by the service is minimal. Inbound and outbound traffic reaches approximately 2 Mbps at around 4000 requests per second – the maximum load I achieved from the pool. UDP packet counts will roughly correspond to request volumes. The pool offers various load levels; choose one matching your capacity.

## Configuration Guide

This guide assumes you understand the NTP protocol, the purpose of pool.ntp.org, how to add your server to it, and how to monitor its status.

**Requirements:**
*   Static public IPv4 and/or IPv6 address.
*   Stable internet connection (24/7 uptime AND consistently low latency).

**1. Firewall Configuration:**

Open UDP port 123 on the WAN interface. Add this rule to `/etc/config/firewall`:
```
config rule
    option src 'wan'
    option name 'ntp'
    option dest_port '123'
    option target 'ACCEPT'
    list proto 'udp'
```

**2. Install chrony and Disable sysntpd:**

Install the `chrony` package. Stop and disable the default `sysntpd` service (`/etc/init.d/sysntpd stop; /etc/init.d/sysntpd disable`). They cannot and should not run concurrently.

**3. Configure chrony:**

`chrony` uses two configuration files:
*   `/etc/config/chrony`: Standard OpenWrt UCI configuration (defines upstream servers).
*   `/etc/chrony/chrony.conf`: Primary `chrony` configuration (additional settings not managed by UCI).

**A. UCI Config (`/etc/config/chrony`):**

Example configuration with several reliable global servers: [chrony](chrony)

**Do not keep these exact servers.** Strongly prefer selecting servers from:
*   [Stratum 2](https://support.ntp.org/Servers/StratumTwoTimeServers)
*   [Stratum 1](https://support.ntp.org/Servers/StratumOneTimeServers)
    
Prioritize geographical proximity for more stable latency. Reliability is more critical than connecting to Stratum 1 (though being Stratum 2 is nice).
    
**Tip:** Initially add servers dynamically (see below) for testing. Only add well-performing servers to this file later.
    
The example config listens on interfaces `lan`, `wan`, and `wan_6`. Modify this list:
*   Remove `lan` if LAN clients shouldn't use this server for sync (requires additional steps later).
*   Remove `wan` or `wan_6` if you lack or don't wish to use that IP version publicly.
*   Adjust for non-standard interface names.

**B. Primary Config (`/etc/chrony/chrony.conf`):**

My modified configuration: [chrony.conf](chrony.conf)

Key differences from the package default:
*   **Client Logging:** `clientloglimit 268435456` (256 MB) replaces `noclientlog`. This stores recent client IPs primarily to enable rate limiting (`ratelimit`) and block excessive queries from misbehaving clients, reducing load. Adjust the size based on available memory.
*   **Process Priority:** `sched_priority 50` increases `chrony`'s scheduling priority (max 99). While `chrony` is lightweight, higher priority may help. No negative impact observed, but experiment if needed.
*   **Drift File Path:** `driftfile /mnt/opt/chrony/drift` points to persistent storage (eMMC). This file stores the measured system clock drift rate. Its persistence across reboots is crucial for fast startup compensation, especially given the router's significant inherent clock drift. Default path (`/var`) is volatile. Writing this small file periodically is unlikely to harm a typical 8GB eMMC. Revert if concerned.
*   **Statistics Dump Directory:** `dumpdir /mnt/opt/chrony` stores servers state statistics on shutdown for faster restart synchronization. To enable loading this state, modify `/etc/init.d/chronyd`: find `procd_set_param command $PROG -n` and change to `procd_set_param command $PROG -n -r`
**Note:** Dumps only occur on clean shutdown. Add a cron job for periodic dumps:
`*/15 * * * * /usr/bin/chronyc dump` (Dumps every 15 minutes. Adjust interval or omit entirely if concerned about eMMC wear).

**4. Conntrack Tuning:**
High query volume creates significant `conntrack` (connection tracking) load ("Active connections" in LuCI). Default limits (65536 connections) will be exceeded at pool connection speed setting >=1GBit, causing dropped requests.

**Solutions:**
*   **Option 1 (Increase Limit):** Create [/etc/sysctl.d/12-nf_conntrack_max.conf](12-nf_conntrack_max.conf) and reboot router. To apply immediately call:
```
sysctl -w net.netfilter.nf_conntrack_max=262144
sysctl -w net.netfilter.nf_conntrack_udp_timeout=30`
```
*   **Option 2 (Disable Tracking for NTP packets):** Preferable here.

Create [/etc/nftables.d/10-ntp-notrack.nft](10-ntp-notrack.nft). Apply: `/etc/init.d/firewall restart`.
This disables pointless tracking for UDP/123 traffic and allows connection statistics using these rules (see below).

**5. Start chronyd:**

Start and enable the service: `/etc/init.d/chronyd start; /etc/init.d/chronyd enable`.

When adding your server to the pool, do not start with the maximum load. Begin with the lowest available setting (currently 512 Kbit). Only after confirming stable operation should you gradually increase the load.

I recommend waiting until the pool assigns your server a score of 20.0 before raising the load. This score serves as a reliable external indicator of successful operation.

Important note about load values:
The numbers shown in the dropdown menu (e.g., 512 Kbit, 1 Gbit) do not directly represent actual traffic volume. However, they are useful for estimating relative load increases when moving to the next tier.

This gradual approach proved valuable in my case: it allowed me to detect the connection limit issue early and implement the necessary fixes before scaling further.

## Monitoring Sources & Dynamic Management
Monitor upstream server status immediately using:
   ```
   chronyc -N sources
   chronyc -N sourcestats
   ```
*   **`sources`:** Focus on the leftmost column. `^*` indicates the current sync source. `^+` indicates candidates. `^-` are tracked but unused. `^?` indicates unreliability. A stable `Reach` value of `377` indicates reliable syncs.
*   **`sourcestats`:** Key columns: `Offset` (time difference), `Std Dev` (measurement dispersion). `chrony` selects servers with low dispersion and consistent offset.

**Dynamically Manage Servers:**
*   Add: `chronyc add server <address>`
*   Delete: `chronyc delete <address>`
    **Crucial:** Use the IP address (`chronyc -n sources` output, note flag `-n`) or the *reverse DNS name* (`chronyc sources` output *without* flags) for deletion. The name used during addition (`-N` flag output) often doesn't work for deletion due to reverse lookups.
*   Force Reselection: If a good server isn't selected for period of time: `chronyc reselect`. Servers will temporarily show `^-` before `^*`/`^+` reappear.

Changes made with `chrony add`/`chrony delete` are volatile and lost on service restart.

Once stable servers are identified via dynamic testing, add them to `/etc/config/chrony` using the example format. Add `option iburst 'yes'` to 1-2 primary servers for faster initial sync. Avoid enabling `iburst` for all servers.

`chronyc tracking` provides detailed information about the local clock's synchronization status. Consult `chrony` documentation for parameter explanations.

## DHCP Advertisement (LAN Clients)
To advertise this server to LAN clients via DHCP, modify `/etc/config/dhcp`:
*   IPv4: Add `list dhcp_option '42,192.168.1.1'` (Use your router's LAN IP) to the `config dhcp 'lan'` section.
*   IPv6: Add `list dhcp_option 'ntp,fd19:8fa6:8bee::1'` (Use your router's LAN IPv6 ULA/GUA) to the same section.

Note: Many devices ignore DHCP-advertised NTP servers, relying on hardcoded addresses. Inclusion is generally harmless.

## Statistics Collection & Visualization
I implemented a basic statistics system using `rrdtool1`.

**Components:**

[/usr/bin/ntp-stats-init](ntp-stats-init): Initializes the RRD database. Adjust `RRA:AVERAGE:0.5:1:4320` (last number = samples stored). 4320 samples (1/minute) = ~3 days data (~600 KB). Modify for longer retention.

[/usr/bin/ntp-monitor](ntp-monitor): Collects data every minute:
*   CPU % (from `/proc/PID/stat`)
*   Requests (from `chronyc serverstats`)
*   Traffic In/Out (using counters from the `nftables` rules described above)
*   Time Status (from `chronyc tracking`)

[/usr/bin/ntp-plot](ntp-plot): Generates graphs using `rrdtool`. Called by `ntp-monitor` (set `PLOT_INTERVAL=1` in `ntp-monitor` to plot every minute). Specify output directory and time range (e.g., 24h, 72h).

[/www/stats.html](stats.html): Example page displaying the graphs. Symlink generated PNGs into `/www` for browser access via LuCI/uHTTPd.

**Persistence:**
*   **Cron Backup:** Add to crontab: `0 1 * * * /bin/cp /tmp/stats/ntp_stats.rrd /mnt/opt/chrony` (Adjust path/interval).
*   **Restore on Boot:** Add to `/etc/rc.local`:
```
mkdir -p /tmp/stats
[ -f /mnt/opt/chrony/ntp_stats.rrd ] && cp /mnt/opt/chrony/ntp_stats.rrd /tmp/stats/
```