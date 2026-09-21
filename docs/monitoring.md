# Monitoring

Monitoring for ABRVN Homelab is built around **Webmin** and a custom Webmin module called **Webmin Homelab Monitor**.

The monitor began as an ABRVN-specific dashboard for quickly checking the health of the server. It was later separated into its own reusable open-source project so that the monitoring code could be maintained independently from both Webmin itself and the private homelab configuration.

> **Project:** [Webmin Homelab Monitor](https://github.com/Ravi2k71/webmin-homelab-monitor)  
> **Live demo:** [GitHub Pages demo](https://ravi2k71.github.io/webmin-homelab-monitor/)  
> **License:** GNU GPL v3.0

The public demo uses fictional data and is not connected to the ABRVN server.

---

## Monitoring Goals

The monitoring layer is intended to answer a few basic operational questions from one place:

- Is the host healthy?
- Are storage and RAID functioning normally?
- Are important containers and services running?
- Is remote connectivity working?
- Is the firewall in the expected state?
- Did the scheduled backup system run successfully?
- Are there obvious network or disk problems that need attention?

The goal is not to replace a full metrics stack such as Prometheus and Grafana. For this homelab, the dashboard provides a lightweight operational overview directly inside the administration interface already used to manage the server.

---

## Current Monitoring Coverage

The current monitor can display:

| Area | Monitoring |
| --- | --- |
| System | CPU utilization, memory utilization, GPU utilization, uptime |
| Overall health | Combined whole-server health summary |
| Storage | Filesystem capacity and utilization |
| RAID | Linux software RAID / mdadm state |
| Drives | SMART health information |
| Containers | Docker container status |
| Remote access | Tailscale state and Funnel detection |
| Security | UFW firewall state and SSH exposure summary |
| Backups | systemd backup timer/service status |
| Network | Interface state, LAN address, gateway reachability, Internet connectivity |
| Diagnostics | DNS resolution, latency, packet loss |
| Optional data | Open-Meteo weather information |

Collectors that are not relevant to a deployment can be disabled rather than being treated as failures.

---

## Architecture

The monitor is implemented as a standalone Webmin module rather than by modifying Webmin's core files.

```text
                         Webmin
                           │
                           ▼
                Webmin Homelab Monitor
                           │
          ┌────────────────┼────────────────┐
          │                │                │
          ▼                ▼                ▼
      Host data        System tools      Services
     /proc + /sys       / commands      / applications
          │                │                │
          └────────────────┼────────────────┘
                           ▼
                    Collector results
                           │
                           ▼
                  Overall health view
```

The module is written primarily in Perl/CGI and gathers information from Linux system interfaces and installed command-line tools. Depending on the enabled collectors, this can include utilities such as `mdadm`, `smartctl`, Docker, Tailscale, UFW, systemd, `ip`, `ping`, and DNS-resolution tools.

Keeping the module separate from Webmin core files is intentional. It reduces the chance that a Webmin update will overwrite custom monitoring work and makes the project easier to package, test, and reuse.

---

## Configuration

The public project separates machine-specific settings from the reusable dashboard logic.

Configuration includes items such as:

- Dashboard branding
- Storage mount points
- RAID array and member devices
- SMART devices
- Network interface and gateway
- Connectivity-test targets
- Backup timer and service
- Weather location
- Collector enable/disable settings

The packaged module supports Webmin's native **Module Config** interface and safe defaults for fresh installations.

This separation is also important for privacy: public source code and examples do not need to contain the ABRVN server's real private addresses, storage paths, or other machine-specific identifiers.

---

## Overall Health

One of the primary goals of the dashboard is to turn multiple individual checks into a quick server-health summary.

Individual collectors report the state of the resources they monitor. Optional collectors can be disabled when they do not apply to a system; a deliberately disabled collector should not make the overall server state appear unhealthy.

This allows the same dashboard design to be adapted to different servers without requiring every supported technology to be installed.

---

## Storage and RAID Monitoring

Storage is especially important in ABRVN Homelab because the primary photo library depends on local disk storage.

The monitor provides visibility into:

- Filesystem utilization
- mdadm RAID state
- RAID member information
- SMART drive health

The dashboard is useful for routine visibility, but it does not change the underlying storage design: monitoring can report a failure or warning, but it does not provide redundancy or replace backups.

---

## Docker and Service Monitoring

The Docker collector provides a quick view of container state so failures in application services are visible without first opening a terminal and manually checking Docker.

For ABRVN Homelab, container monitoring is particularly useful for the PhotoPrism stack and its supporting services.

Backup monitoring uses systemd timer/service state to show whether the scheduled backup infrastructure is operating as expected. This complements backup validation and disaster-recovery work; a successful timer is evidence that a job ran, not proof by itself that every backup is restorable.

---

## Network and Remote-Access Monitoring

The network section combines several small checks rather than reducing connectivity to a single ping.

Current diagnostics can include:

- Network-interface state
- LAN address
- Default-gateway connectivity
- Internet connectivity
- DNS resolution
- Latency
- Packet loss
- Tailscale state
- Tailscale Funnel detection

This is useful because different failures can have similar symptoms. A host may have an active interface while DNS is broken, or have LAN connectivity while an Internet path is unavailable.

---

## Security Monitoring

The dashboard includes visibility into UFW and SSH exposure so that important host-access controls can be reviewed alongside normal system health.

This monitoring is deliberately read-oriented. Firewall and remote-access configuration should still be changed through the appropriate administrative tools rather than automatically altered because a dashboard check reports an unexpected state.

Webmin itself remains an administrative interface and should not be exposed casually. In the ABRVN design, Webmin access is restricted to trusted LAN/Tailscale paths and Webmin's referer/CSRF protections remain enabled.

---

## Open-Source Project

The reusable monitor is maintained separately at:

**[Ravi2k71/webmin-homelab-monitor](https://github.com/Ravi2k71/webmin-homelab-monitor)**

The project includes installation, configuration, customization, and collector documentation; example configurations; an installable Webmin module; and a static GitHub Pages demonstration.

The first installable public release is **Webmin Homelab Monitor v1.0.1**, distributed as a `.wbm.gz` Webmin module package through GitHub Releases.

Separating this project from the main ABRVN Homelab repository has two benefits:

1. The homelab repository can document how monitoring fits into the overall infrastructure.
2. The monitor repository can remain reusable for people whose hardware, network, storage, and services differ from ABRVN.

---

## Security and Privacy

Monitoring software can expose more infrastructure information than expected. Public documentation and example configurations therefore need to be sanitized carefully.

The public project should not contain production passwords, API tokens, private keys, private addresses, sensitive hostnames, private storage paths, or other credentials and identifiers.

The static demo intentionally uses fictional information and does not communicate with the real homelab, Webmin instance, Docker environment, Tailscale network, or storage devices.

---

## Future Monitoring Work

The current dashboard is intended to remain extensible. Potential future collectors and improvements include:

- CPU and disk temperatures
- NVMe health and wear information
- UPS and power status
- Certificate expiration
- Available package updates
- Reboot-required state
- Failed SSH/Webmin authentication visibility
- Network throughput and link information
- Additional filesystem/storage technologies such as ZFS or Btrfs
- Virtual-machine or application-specific health checks
- Historical trends and alerting

These are roadmap ideas rather than claims about the current production deployment.

---

## Design Lessons

The monitoring project reinforced several principles used elsewhere in ABRVN Homelab.

**Keep custom code separate from upstream software.** Modifying Webmin core files would make upgrades harder and could cause custom work to be overwritten.

**Configuration should be portable.** Separating environment-specific values from collector logic makes the project reusable and reduces the chance of publishing private infrastructure details.

**Optional functionality should fail gracefully.** A server that does not use RAID, Docker, Tailscale, weather, or another optional collector should not automatically be classified as unhealthy.

**Monitoring is not recovery.** Knowing that a disk, service, or backup job has failed helps with detection, but recovery procedures and tested backups are still separate requirements.

**A useful dashboard should explain where to look next.** The purpose of the monitor is to make problems visible quickly and reduce the number of separate commands needed for routine health checks.
