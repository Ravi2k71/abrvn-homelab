# Troubleshooting and Lessons Learned

ABRVN Homelab was not built as a clean, one-pass deployment. The current architecture is the result of failed approaches, hardware loss, operating-system problems, networking constraints, storage mistakes, service conflicts, and security changes that had to be tested carefully.

This document records the important incidents and the troubleshooting methods that came out of them.

> Production addresses, credentials, private identifiers, personal data, and sensitive filesystem details are intentionally omitted.

---

## Why Document Failures?

A working diagram shows what the system looks like now. It does not show why the system looks that way.

Several of the strongest design decisions in the current homelab came directly from failures:

- The original server died.
- Disaster recovery was not mature enough when it happened.
- A later operating-system upgrade partially failed and left the system in a mixed state.
- Traditional remote-access plans failed because of ISP-side CGNAT.
- Reverse-proxy and port assumptions had to be redesigned.
- Storage and mount problems affected applications.
- Broad Docker security exceptions turned out to be unnecessary.
- Administration and application access had to be separated more deliberately.

The goal of documenting these events is not to present every mistake as a success. It is to show how the system changed after evidence demonstrated that the original approach was insufficient.

---

# Major Disaster 1 — Loss of the Original Server

One of the most important events in the history of the homelab happened before the current server architecture existed.

The original **Dell OptiPlex 9020** had operated as the server for roughly four years before the machine failed.

The hardware failure itself was only part of the problem.

The larger problem was that disaster-recovery preparation was not mature enough to make rebuilding the environment straightforward.

## Impact

The server had accumulated useful services and configuration over years of operation. When the hardware failed, recovery exposed how much of the environment depended on knowledge and configuration that had not been sufficiently formalized.

The project was ultimately shelved for a period rather than immediately restored into an equivalent production state.

This became the first major lesson that **long uptime is not evidence of recoverability**.

A server can operate successfully for years while still having an untested recovery process.

## What the incident exposed

The failure highlighted several weaknesses:

- Recovery procedures were not sufficiently documented.
- Rebuilding the environment depended too much on remembered configuration.
- Backup and disaster recovery had not been treated as separate engineering requirements.
- Hardware failure had not been rehearsed as a realistic scenario.
- The difference between storage configuration and independent backup protection had not been given enough operational weight.

## Architectural consequences

The current homelab is deliberately being documented in a way the earlier environment was not.

Important changes in philosophy include:

```text
Old approach
    │
    ▼
Keep the server running
    │
    ▼
Assume recovery can be figured out later


Current approach
    │
    ▼
Document architecture
    │
    ├── Back up persistent state
    ├── Back up important configuration
    ├── Automate backup jobs
    ├── Monitor backup status
    ├── Separate storage from backup
    └── Develop/test restore procedures
```

The current disaster-recovery work is still intentionally marked as incomplete because the lesson from the original server is that merely having backup files is not enough.

## Lesson

> **Reliability and recoverability are different properties.**

A server that has not failed recently may appear reliable, but the quality of its recovery plan is unknown until the plan is documented and tested.

---

# Major Disaster 2 — Partial Pop!_OS Upgrade Failure

A second major incident occurred during the current server generation when the system was upgraded from **Pop!_OS 22.04 to Pop!_OS 24.04**.

The upgrade did not complete cleanly.

Instead of producing a simple success or failure, it left the server in a partially upgraded state.

## Symptoms

The recovery process had to deal with a combination of problems rather than one isolated failure.

Observed areas of trouble included:

- Mixed package state
- Package/dependency problems
- Desktop/session components
- PipeWire/audio-related package state
- DNS/network resolution problems
- Tailscale connectivity problems
- General uncertainty about which infrastructure services had survived the upgrade correctly

This was dangerous because the machine could not simply be judged by whether it booted.

A server can boot while application, networking, package-management, or remote-access layers remain broken.

## Recovery approach

The system was recovered **in place** rather than immediately erased and rebuilt.

Recovery involved repairing the package-management state, correcting package/repository problems, resolving DNS/networking issues, and restoring affected service functionality.

The important part came after the obvious errors were repaired.

The recovery was not considered complete until the surrounding infrastructure was checked.

Validation included areas such as:

- package-management health
- systemd/service health
- Docker
- PhotoPrism
- MariaDB
- Caddy
- Samba
- Tailscale
- RAID/storage
- UFW/firewall state

The purpose was to avoid declaring success after fixing only the first visible symptom.

## Why this incident mattered

The upgrade failure reinforced that a server is a dependency graph.

```text
Operating-system upgrade
        │
        ├── Package state
        ├── Networking
        ├── DNS
        ├── Remote access
        ├── Container runtime
        ├── Storage
        ├── Firewall
        └── Applications
```

A change at the operating-system layer can affect nearly every service above it.

## Architectural consequences

The incident strengthened several operating practices used later in the project:

- Back up before major changes.
- Make controlled changes where possible.
- Validate configuration before restarting services.
- Check dependencies rather than only the reported failure.
- Verify remote access after networking changes.
- Verify storage before blaming an application.
- Inspect service state and logs after recovery.
- Treat post-change validation as part of the change itself.

## Lesson

> **A machine booting successfully does not mean the infrastructure has recovered successfully.**

Recovery needs to be validated layer by layer.

---

# Incident — CGNAT Broke the Original Remote-Access Design

The PhotoPrism project initially explored a conventional Internet-access design using public DNS, Caddy, TLS certificate issuance, and router port forwarding.

The approach assumed inbound Internet traffic could reach the home server.

That assumption turned out to be wrong.

## Symptoms

The expected public HTTP/HTTPS path could not be reached reliably even after local configuration and router forwarding were checked.

ACME HTTP challenge attempts could not complete successfully through the intended public path.

The important clue was the ISP addressing environment.

The connection was behind **carrier-grade NAT (CGNAT)**, which meant normal router port forwarding could not create the expected end-to-end inbound path.

## Failed mental model

```text
Internet
   │
   ▼
Public IPv4
   │
   ▼
Home router port forward
   │
   ▼
Caddy
   │
   ▼
PhotoPrism
```

This model only works when the required inbound connectivity actually reaches the customer's router.

## Resolution

Rather than continuing to modify Caddy, firewall rules, or router forwards for a path the ISP architecture prevented, the project changed the remote-access architecture.

Tailscale became the private remote-administration layer, and **Tailscale Funnel** became the remote application path for PhotoPrism.

```text
Internet
   │
   ▼
Tailscale Funnel
   │
   ▼
Caddy
   │
   ▼
PhotoPrism
```

## Lesson

> **When repeated configuration changes do not fix a network problem, verify the network assumptions themselves.**

The problem was not simply another missing port-forwarding rule. The architecture depended on a capability the Internet connection did not provide.

---

# Incident — Reverse Proxy and Port Conflicts

Caddy is part of the PhotoPrism access path, but reverse-proxy work introduced its own troubleshooting challenges.

At one stage, expected HTTP/HTTPS bindings conflicted with other service/network assumptions.

This reinforced the need to inspect actual listeners rather than assuming a configuration file represents the running network state.

## Troubleshooting method

Useful questions became:

- Which process is actually listening?
- On which address?
- On which port?
- Is the listener on the host or inside Docker?
- Is traffic expected to enter through LAN, Tailscale, Funnel, or Docker networking?
- Is another process already using the port?
- Is the reverse proxy reachable locally before testing the remote path?

The application path should be tested from the inside outward:

```text
PhotoPrism
    │
    ▼
Local application reachability
    │
    ▼
Caddy
    │
    ▼
Local proxy test
    │
    ▼
Tailscale/Funnel
    │
    ▼
Remote client
```

## Lesson

> **Troubleshoot network paths one boundary at a time.**

Testing the entire Internet-to-application path at once makes it difficult to identify which layer failed.

---

# Incident — Storage and Mount Problems

Storage issues during earlier builds demonstrated how a filesystem problem can look like an application problem.

Photo applications and containers depend on host storage being present at the expected location with the expected permissions.

If the underlying filesystem or RAID mount is missing, the container may still start while seeing an empty or incorrect directory.

## Dependency chain

```text
Physical disks
     │
     ▼
mdadm array
     │
     ▼
Filesystem
     │
     ▼
Mount
     │
     ▼
Permissions
     │
     ▼
Docker bind mount
     │
     ▼
Application
```

Troubleshooting from the application downward can waste time if the real problem exists several layers below.

## Lesson

> **Verify the host storage before troubleshooting the application that consumes it.**

For storage-backed services, confirm the array, filesystem, mount, permissions, and bind mount before assuming the application has lost data.

---

# Incident — Earlier Photo Platform Could Not See the Intended Storage

Before the current PhotoPrism deployment, another photo-management approach was attempted.

The intended RAID-backed path was not being detected or used as expected.

Rather than continuing to force an increasingly complicated configuration, the project eventually moved to PhotoPrism.

This was a useful reminder that sunk time is not a reason to keep a poor fit.

## Lesson

> **Changing the solution can be more effective than endlessly repairing the original approach.**

Troubleshooting should determine whether a configuration is wrong, but it should also leave room for the possibility that a different tool better fits the environment.

---

# Incident — Tailscale and Remote-Access Recovery

Remote administration became especially important after the homelab depended on Tailscale for access outside the local network.

When Tailscale connectivity was disrupted during system recovery work, the priority shifted from adding features to restoring a known-good management path.

This led to a broader project decision: **recovery and stability take priority over feature expansion when core access is uncertain.**

## Recovery priorities

```text
Host reachable?
      │
      ▼
Tailscale healthy?
      │
      ▼
SSH reachable?
      │
      ▼
Caddy / Funnel path?
      │
      ▼
PhotoPrism?
      │
      ▼
Webmin / monitoring?
      │
      ▼
Resume feature work
```

## Lesson

> **Restore the management plane before adding new application features.**

When remote administration is unreliable, adding more services increases uncertainty instead of improving the system.

---

# Incident — SSH Firewall Access

SSH troubleshooting demonstrated the difference between a service running and a service being reachable.

An SSH daemon can be healthy while UFW prevents the desired network path from reaching it.

The solution was not to weaken the firewall globally. Access was added only for the intended trusted path.

This later evolved into the current design where SSH uses key authentication and is scoped to trusted LAN/Tailscale connectivity.

## Lesson

> **Reachability and service health are different questions.**

Before changing authentication or reinstalling a service, verify whether packets are allowed to reach it.

---

# Incident — Webmin Access and Browser Security Protections

Webmin troubleshooting produced an important security lesson when browser access encountered referer-related protection.

A tempting workaround would have been to weaken or disable the protection.

That was rejected.

The correct approach was to preserve Webmin's referer/CSRF protections and fix the access/configuration path instead.

## Lesson

> **Do not disable a security control simply because it is the component reporting the problem.**

A security warning can be evidence that the protection is functioning as designed.

---

# Incident — PhotoPrism Container Security Exceptions

The PhotoPrism Compose configuration previously included:

```yaml
security_opt:
  - seccomp:unconfined
  - apparmor:unconfined
```

These settings disabled important default container protections.

The key troubleshooting question was whether PhotoPrism actually required those exceptions.

## Safe testing sequence

The hardening was performed incrementally:

```text
Record baseline
      │
      ▼
Remove seccomp exception
      │
      ▼
Validate Compose
      │
      ▼
Recreate PhotoPrism only
      │
      ▼
Test real workloads
      │
      ▼
Review logs
      │
      ▼
Remove AppArmor exception
      │
      ▼
Repeat validation
```

The final container operated correctly using Docker's normal seccomp filtering and the `docker-default` AppArmor profile while retaining required graphics-device access.

Testing included normal photo browsing, thumbnails, full-resolution images, representative video playback, container inspection, application logs, and host kernel/audit output.

No AppArmor/security-policy denials attributable to the change were found.

## Lesson

> **Security exceptions should be proven necessary, not inherited indefinitely.**

Removing one exception at a time made it possible to identify regressions without changing several security variables simultaneously.

---

# Incident — Video Playback Was Not a Container-Security Failure

After container hardening, some newer HEVC and professional-camera video remained choppy while common photos, thumbnails, older phone videos, and other normal workloads worked.

Because the issue existed independently of the new AppArmor/seccomp posture and there were no corresponding security-policy denials, the evidence did not support treating it as a hardening regression.

Hardware inspection showed the current platform uses an older **Intel HD Graphics 530** integrated GPU.

The project chose not to turn this into a major optimization effort on aging hardware.

## Lesson

> **Do not force every observed problem into the most recent change.**

Temporal proximity is not proof of causation. Logs, reproducibility, hardware capability, and baseline behavior matter.

---

# Incident — Email Alerting Blocked by External Account Requirements

The project planned email-based system alerts using a dedicated Yahoo account and `msmtp`.

Local mail tooling and configuration could be prepared, but the required app-password capability was not available for the account at that time.

Rather than embedding the normal account password or weakening authentication, the notification work was paused.

## Lesson

> **A blocked security prerequisite is a reason to pause a feature, not bypass the prerequisite.**

Monitoring can continue locally until a secure notification path is available.

---

# Troubleshooting Method

The incidents above produced a repeatable diagnostic process.

## 1. Define the symptom precisely

Avoid descriptions such as:

> "The server is broken."

Prefer questions such as:

- Can the host be reached?
- Is DNS resolving?
- Is the service running?
- Is the expected port listening?
- Can the service be reached locally?
- Can it be reached through the proxy?
- Can it be reached through Tailscale?
- Is the storage mounted?
- Did the container start?
- Is the database healthy?

A precise symptom narrows the failure domain.

---

## 2. Identify the layers involved

For a remote PhotoPrism problem:

```text
Client
  │
  ▼
Internet / remote network
  │
  ▼
Tailscale Funnel
  │
  ▼
Caddy
  │
  ▼
Docker networking
  │
  ▼
PhotoPrism
  │
  ├──► MariaDB
  └──► Storage
```

Testing each boundary is more useful than repeatedly changing the final application.

---

## 3. Inspect before changing

Before editing configuration, collect the current state.

Depending on the problem this can include:

- service status
- Docker container state
- listening sockets
- firewall state
- mount state
- RAID state
- SMART information
- Tailscale state
- recent logs
- effective application/container configuration

This preserves evidence and reduces guesswork.

---

## 4. Back up important configuration

Before a risky change, preserve the known state when practical.

A backup provides both a rollback path and a reference for understanding what changed.

---

## 5. Change one variable

Changing the firewall, proxy, application, DNS, and Docker configuration simultaneously may make a problem disappear, but it does not establish which change fixed it.

Small changes improve diagnosis.

---

## 6. Validate configuration before restart

Where a tool supports configuration validation, use it before restarting or recreating a working service.

A syntax error should be discovered before it becomes an outage.

---

## 7. Restart only what is required

If only PhotoPrism changed, recreating the entire stack adds unnecessary variables.

The same principle applies to host services.

---

## 8. Test real behavior

A process showing `running` is useful information, but it is not the same as the service working for a user.

Validation should include the actual workflow:

- login
- browse photos
- load full-resolution media
- play representative video
- reach the service remotely
- use SSH
- use Webmin
- read/write storage where appropriate

---

## 9. Review logs after the test

Logs are most useful when correlated with a known test.

For example:

```text
Perform test
     │
     ▼
Observe failure/success
     │
     ▼
Immediately inspect relevant logs
```

This is more useful than searching a large log history without knowing when the event occurred.

---

## 10. Verify adjacent dependencies

After a major operating-system or network recovery, check more than the service that originally failed.

A repaired DNS problem may still leave Tailscale broken. A repaired package state may still leave Docker unhealthy.

Recovery should verify the dependency graph.

---

# Common Diagnostic Order

For a service that appears unavailable, a useful general order is:

```text
1. Host alive?
2. Storage healthy and mounted?
3. Network interface healthy?
4. DNS working?
5. Firewall allowing intended path?
6. Service/container running?
7. Port/listener correct?
8. Internal dependency healthy?
9. Reverse proxy healthy?
10. Remote-access layer healthy?
11. Authentication working?
12. Logs clean after real test?
```

The exact order can change depending on the symptom, but the principle is to move from foundational dependencies toward the user-facing application.

---

# Lessons That Changed the Architecture

The project now follows several practices specifically because earlier approaches failed.

**Recoverability is designed, not assumed.**  
The loss of the original server demonstrated why backups, rebuild documentation, and restore testing need to exist before a disaster.

**Major upgrades are infrastructure events.**  
The partial Pop!_OS upgrade showed that package, network, container, storage, firewall, and application layers all need post-upgrade validation.

**Network architecture starts with the actual ISP path.**  
CGNAT made traditional inbound forwarding unsuitable, leading to the Tailscale/Funnel design.

**Private administration stays private.**  
Remote application access does not justify public SSH or Webmin exposure.

**Storage is checked before applications.**  
Applications cannot correctly consume data from a missing or incorrect host mount.

**Security controls are not disabled casually.**  
Webmin protections were preserved, and unnecessary Docker security exceptions were removed rather than normalized.

**One change at a time improves evidence.**  
Incremental hardening made it possible to determine that Docker default protections did not break PhotoPrism.

**Logs should confirm a hypothesis, not replace one.**  
Troubleshooting starts by identifying the failure boundary and then using logs to test that explanation.

**Old hardware changes optimization priorities.**  
Not every compatibility issue deserves extensive tuning when a future platform migration will change the constraint.

---

# Relationship to Disaster Recovery

Troubleshooting and disaster recovery overlap but are not the same.

Troubleshooting attempts to restore a malfunctioning system while preserving the current environment where practical.

Disaster recovery assumes that some part of the current environment may no longer be usable.

```text
Problem
  │
  ├── Can current system be repaired safely?
  │          │
  │          └──► Troubleshooting
  │
  └── Is rebuild/restore required?
             │
             └──► Disaster Recovery
```

The original OptiPlex failure and the partial operating-system upgrade illustrate both sides of that distinction.

See [Backup and Disaster Recovery](backup-recovery.md).

---

# What Still Needs Better Documentation

Future incident documentation should become more structured.

A useful incident record should contain:

```text
Date
Impact
Symptoms
Known-good state
Changes immediately before failure
Evidence collected
Root cause
Recovery steps
Validation performed
Preventive action
Remaining risk
```

This will make future troubleshooting less dependent on memory and make the repository a more useful record of how the infrastructure evolves.

---

# Related Documentation

- [Architecture](architecture.md)
- [Backup and Disaster Recovery](backup-recovery.md)
- [Security](security.md)
- [Networking](networking.md)
- [Docker](docker.md)
- [Operating System](operating-system.md)
- [PhotoPrism](photoprism.md)
- [Monitoring](monitoring.md)

The goal of these documents together is to describe not only the final architecture, but the operational reasoning that produced it.
