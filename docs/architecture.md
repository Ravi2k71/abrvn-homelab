# Architecture

ABRVN Homelab is a self-hosted Linux infrastructure environment built around a repurposed workstation. The architecture separates the public application path, internal application services, storage, administration, monitoring, and recovery responsibilities rather than treating the server as one monolithic service.

The primary production workload is **PhotoPrism CE**, but the project is also a hands-on environment for Linux administration, containerization, networking, security hardening, monitoring, backup/recovery, and infrastructure documentation.

> This document intentionally uses generalized architecture and omits private production addresses, credentials, account information, and sensitive filesystem locations.

---

## Design Goals

The architecture is guided by several principles:

- **Least necessary exposure** — only services with a real access requirement should be reachable.
- **Separation of responsibilities** — application, database, administration, storage, and monitoring functions should remain logically distinct.
- **Private administration** — SSH and Webmin should not share the public application access path.
- **Container isolation** — internal services should communicate through Docker networking instead of unnecessary host ports.
- **Defense in depth** — firewalling, authentication, container confinement, and private networking complement one another.
- **Recoverability** — backups and disaster-recovery planning are separate requirements from storage availability.
- **Maintainability** — custom monitoring and configuration should survive upstream software updates.
- **Documentation before complexity** — infrastructure decisions should be understandable and reproducible.

---

## High-Level Architecture

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
                          /           \
                         /             \
                        ▼               ▼
                Photo Originals      MariaDB
                        │
                        ▼
                   RAID Storage


                     Administration
                          │
                       Tailscale
                      /         \
                     ▼           ▼
                    SSH        Webmin
                                  │
                                  ▼
                         Homelab Monitor


                    Recovery Layer
                          │
                          ▼
                Automated Backups
                          │
                          ▼
             Restore / DR Development
```

The public and administrative paths are deliberately different. PhotoPrism is the service that requires remote application access, while administrative interfaces remain restricted to trusted connectivity.

---

## Physical Host

The current server is a **Dell Precision 3620** running **Pop!_OS 24.04 LTS**.

Current platform:

| Component | Implementation |
| --- | --- |
| CPU | Intel Core i5-6500 |
| Memory | 16 GB RAM |
| Graphics | Intel HD Graphics 530 |
| Operating System | Pop!_OS 24.04 LTS |
| Storage array | 2 × 1 TB HDD |
| RAID | Linux mdadm RAID 0 |
| Container runtime | Docker / Docker Compose |

The current platform is intentionally modest. Part of the project is learning how to build a useful service stack with existing hardware before moving to a newer platform.

See [Hardware](hardware.md) and [Operating System](operating-system.md).

---

## Application Layer

### PhotoPrism

PhotoPrism is the primary user-facing application. It provides the photo library interface, indexing, metadata handling, authentication, and photo/video browsing.

It runs as a Docker container rather than being installed directly onto the host operating system.

Important architectural choices include:

- Originals are host-mounted rather than stored inside the container filesystem.
- Application storage is separated from the photo originals.
- MariaDB runs as a separate service.
- PhotoPrism does not need to expose every internal service directly to the LAN.
- Remote application traffic reaches PhotoPrism through the reverse-proxy path.

See [PhotoPrism](photoprism.md).

### MariaDB

MariaDB provides the application database for PhotoPrism.

The database is an internal dependency, not a user-facing service. It communicates with PhotoPrism through Docker networking and does not require direct Internet exposure.

Separating the database from the application container makes the dependency explicit and allows the database lifecycle and data storage to be handled independently.

---

## Container Layer

Docker and Docker Compose provide the application runtime.

Current containerized services include:

- PhotoPrism
- MariaDB
- Caddy

Docker networks are used to separate internal application communication from host-network exposure.

Conceptually:

```text
                   Host
                    │
        ┌───────────┴───────────┐
        │                       │
   Proxy path              Internal path
        │                       │
        ▼                       ▼
      Caddy ───────────────► PhotoPrism
                                 │
                                 ▼
                              MariaDB
```

Only traffic that needs to leave the container environment should require an appropriate host or proxy path.

### Container security

PhotoPrism previously used broad `seccomp:unconfined` and `apparmor:unconfined` exceptions. Testing demonstrated that those exceptions were unnecessary.

The current deployment uses Docker's normal seccomp filtering and the `docker-default` AppArmor profile while retaining the required Intel graphics device access. Application behavior and logs were validated after the hardening change.

This is an example of a general architecture rule used throughout the project: security exceptions should exist only when there is a demonstrated technical requirement.

See [Docker](docker.md).

---

## Reverse Proxy and Public Application Path

Caddy provides the reverse-proxy layer between the remote access path and PhotoPrism.

The logical request flow is:

```text
Remote client
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

Caddy provides a controlled application entry point rather than requiring PhotoPrism itself to become the general-purpose network edge.

The project originally explored conventional inbound port forwarding and direct public reverse-proxy access. ISP-side networking limitations made that approach impractical, which led to the current Tailscale-based design.

See [Networking](networking.md).

---

## Tailscale and Remote Connectivity

Tailscale has two different roles in the architecture.

### Private administration

Trusted devices use Tailscale to reach administrative services such as SSH and Webmin without exposing those interfaces directly to the public Internet.

### PhotoPrism access

Tailscale Funnel provides the remote application path used to reach the Caddy/PhotoPrism stack.

These two roles should not be confused. The existence of a public application path does not mean administrative services are intended to be public.

---

## Host Security Boundary

UFW provides host-level firewall policy.

The firewall complements rather than replaces other controls:

```text
Network reachability
       │
       ▼
      UFW
       │
       ▼
Service binding / Docker exposure
       │
       ▼
Application authentication
       │
       ▼
Container / OS security controls
```

Administrative access is additionally hardened through SSH public-key authentication, disabled SSH password authentication, disabled direct root SSH login, restricted Webmin access, and PhotoPrism multi-factor authentication.

Docker's seccomp/AppArmor protections add another boundary around containerized workloads.

No single control is treated as sufficient by itself.

---

## Storage Architecture

The current photo storage resides on a Linux software RAID array.

```text
        Physical Disk A
              \
               \
                ► mdadm RAID 0 ► Filesystem ► Photo Library
               /
              /
        Physical Disk B
```

RAID 0 provides combined capacity but no redundancy. Failure of either member can result in loss of the array.

For this reason, the architecture explicitly separates **storage** from **backup**.

> RAID is not considered a backup mechanism.

A future platform/storage migration is expected to prioritize redundancy for important data rather than continuing to rely on RAID 0.

---

## Backup and Recovery Layer

Automated backups protect important PhotoPrism data and supporting configuration.

The recovery architecture is conceptually separate from the live service:

```text
              Production services
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
     Application data    Configuration
          │                   │
          └─────────┬─────────┘
                    ▼
             Backup process
                    │
                    ▼
             Backup storage
                    │
                    ▼
          Restore / DR testing
```

The existence of a successful backup job is not treated as proof of recoverability. Restore procedures, restore testing, hardware-failure planning, and stronger off-system/off-site protection remain part of the disaster-recovery roadmap.

---

## Administration Layer

Two primary administration interfaces are used:

### SSH

SSH provides terminal-level administration and is configured around key-based authentication. Direct root login and password authentication are disabled.

### Webmin

Webmin provides browser-based server administration. It is restricted to trusted network paths rather than being publicly exposed.

Webmin's normal referer and CSRF protections remain enabled.

The two interfaces provide different administrative workflows, but both are treated as privileged management surfaces.

---

## Monitoring Layer

The server uses **Webmin Homelab Monitor**, a custom Webmin module developed from the ABRVN environment and later separated into its own reusable open-source project.

The monitoring layer gathers information from the host and supporting tools to provide visibility into:

- CPU, memory, GPU, and uptime
- Storage utilization
- RAID state
- SMART disk health
- Docker containers
- Tailscale and Funnel
- UFW and SSH exposure
- Backup jobs
- Network state and connectivity
- DNS, latency, and packet loss
- Overall server health

Monitoring is intentionally separated from Webmin core code so custom functionality is less likely to be overwritten by Webmin updates.

See [Monitoring](monitoring.md).

---

## Trust Boundaries

The architecture can be viewed as several trust zones.

### Public / untrusted

Internet-originated application traffic should reach only the intended public application path.

### Trusted remote devices

Authorized devices connected through the private Tailscale network can reach approved administrative services.

### Host

The Linux host controls local storage, firewalling, container runtime, system services, and privileged administration.

### Container networks

Application services communicate through Docker networks. Internal dependencies such as MariaDB do not need the same exposure as the user-facing application.

### Storage and backups

Production storage and recovery copies serve different purposes and should not be treated as one failure domain.

This model helps determine where a service belongs before deciding how it should be exposed.

---

## Failure Domains

A useful architecture document should describe not only normal traffic flow but also what can fail.

### Internet or ISP failure

Local services may remain functional while remote access becomes unavailable.

### Tailscale/Funnel failure

The server and local PhotoPrism deployment may remain healthy even if the remote access path is unavailable.

### Caddy failure

PhotoPrism may remain running internally while the normal reverse-proxy path fails.

### PhotoPrism failure

The database, storage, host, and remote network can remain operational even when the application container is unhealthy.

### MariaDB failure

PhotoPrism can become unusable even though the web container itself is running.

### RAID member failure

Because the current array is RAID 0, one disk failure can affect the entire photo-storage array.

### Boot-drive or host failure

Container configuration and photo data may survive on separate storage, but full recovery depends on documented backups and restoration procedures.

Thinking in failure domains is one reason disaster recovery remains a separate project phase rather than being considered complete when the service is merely online.

---

## Data Flow

A simplified photo request follows this path:

```text
Client
  │
  ▼
Remote access layer
  │
  ▼
Caddy
  │
  ▼
PhotoPrism
  │
  ├────────► MariaDB
  │
  └────────► Photo storage
```

Administrative traffic follows a different path:

```text
Trusted administrator device
            │
            ▼
         Tailscale
          /     \
         ▼       ▼
       SSH     Webmin
                  │
                  ▼
              Monitoring
```

Separating these flows reduces the need to expose management services simply to make the application remotely usable.

---

## Architecture Evolution

The current architecture is the result of several iterations rather than a single initial design.

Major changes have included:

1. Moving from conventional public port-forwarding plans to Tailscale-based remote connectivity.
2. Separating the database and application through Docker networking.
3. Tightening host firewall and administrative access.
4. Moving SSH administration to key-based authentication.
5. Adding PhotoPrism multi-factor authentication.
6. Removing unnecessary unconfined seccomp/AppArmor container settings.
7. Building dedicated backup automation.
8. Developing monitoring as a modular Webmin project instead of modifying Webmin core files.
9. Expanding the repository from setup notes into architecture, security, operations, and recovery documentation.

The architecture is expected to continue changing as storage becomes redundant, disaster-recovery testing matures, monitoring improves, and the server eventually moves to newer hardware.

---

## Current Limitations and Planned Improvements

Known architectural limitations include:

- RAID 0 provides no storage redundancy.
- Disaster-recovery restore procedures still need additional validation and formal testing.
- Off-system/off-site backup resilience needs further development.
- Monitoring is primarily a current-state operational view rather than a historical metrics platform.
- The current hardware platform limits the value of spending significant effort on advanced media acceleration.
- Additional network segmentation and access-policy refinement may be useful as the environment grows.

These limitations are documented deliberately so the repository does not present roadmap work as completed infrastructure.

---

## Related Documentation

- [Hardware](hardware.md)
- [Operating System](operating-system.md)
- [Docker](docker.md)
- [Networking](networking.md)
- [PhotoPrism](photoprism.md)
- [Monitoring](monitoring.md)

Backup/recovery, security, and troubleshooting documents will be added as those areas are formalized for public documentation.
