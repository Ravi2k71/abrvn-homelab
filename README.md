# ABRVN Homelab — Self-Hosted Infrastructure & Security Lab

A practical self-hosted infrastructure project focused on Linux administration, containerization, storage, networking, secure remote access, monitoring, automation, and disaster recovery.

The project began as a family photo server and grew into a hands-on environment for learning systems administration and cybersecurity while operating real services.

> **Status:** Active development — core PhotoPrism infrastructure is operational; documentation and disaster-recovery work are ongoing.

---

## Overview

ABRVN Homelab runs on a repurposed Dell Precision 3620. Its primary production workload is **PhotoPrism CE**, backed by MariaDB and RAID storage. Caddy provides the reverse-proxy layer, Tailscale provides remote connectivity, and Webmin plus a custom monitoring module provide administration and visibility.

The project has two goals:

1. Run useful self-hosted services reliably.
2. Document the engineering decisions, security controls, failures, recovery work, and lessons learned while building them.

This repository intentionally focuses on architecture and sanitized examples rather than publishing private production configuration.

---

## Current Infrastructure

| Component | Current implementation |
| --- | --- |
| Server | Dell Precision 3620 |
| CPU | Intel Core i5-6500 |
| Memory | 16 GB RAM |
| Graphics | Intel HD Graphics 530 |
| Operating System | Pop!_OS 24.04 LTS |
| Containers | Docker / Docker Compose |
| Photo Management | PhotoPrism CE |
| Database | MariaDB |
| Reverse Proxy | Caddy |
| Remote Access | Tailscale / Tailscale Funnel |
| File Sharing | Samba |
| Firewall | UFW |
| Administration | SSH + Webmin |
| Storage | 2 × 1 TB HDD |
| RAID | mdadm RAID 0 |
| Backups | Automated application/configuration backups + separate backup strategy |
| Monitoring | Webmin + Webmin Homelab Monitor |

---

## Architecture

```text
                         Internet
                            │
                    Tailscale Funnel
                            │
                            ▼
                         Caddy
                            │
                            ▼
                       PhotoPrism
                      /          \
                     ▼            ▼
              Photo Storage     MariaDB
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
```

Administrative services are kept separate from the public PhotoPrism path. SSH and Webmin are restricted to trusted network paths rather than being directly exposed to the Internet.

For more detail, see the documents in [`docs/`](docs/).

---

## PhotoPrism

PhotoPrism is the primary application and is deployed with Docker Compose alongside MariaDB.

Current design highlights include:

- Host-mounted photo storage rather than storing originals inside the container
- Separate MariaDB service
- Caddy reverse proxy
- Remote access through Tailscale Funnel
- Multi-factor authentication configured for PhotoPrism accounts
- Scheduled indexing appropriate to the library's upload pattern
- Intel `/dev/dri` device access for supported media workloads
- Automated backups of important application data and configuration

### Container hardening

PhotoPrism previously used the broad Docker security overrides `seccomp:unconfined` and `apparmor:unconfined`. These were removed after testing showed they were unnecessary.

The deployment now uses Docker's normal seccomp filtering and the `docker-default` AppArmor profile while retaining required `/dev/dri` access. Photo browsing, thumbnails, full-resolution images, and supported video playback were validated after the change, and application/kernel logs were checked for security-policy denials.

See [PhotoPrism documentation](docs/photoprism.md) and [Docker documentation](docs/docker.md).

---

## Networking & Remote Access

The original design explored traditional inbound port forwarding and public reverse-proxy access. ISP-side networking constraints made that design impractical, leading to a Tailscale-based architecture.

The current design uses:

- **Tailscale** for remote administration
- **Tailscale Funnel** for the PhotoPrism web application
- **Caddy** as the application reverse proxy
- **UFW** for host-level access restrictions
- **Docker networks** for service isolation

The guiding principle is **least necessary exposure**: a service is reachable only where there is a specific operational requirement.

See [Networking documentation](docs/networking.md).

---

## Security

Security controls currently include:

- UFW with service- and network-specific rules
- SSH public-key authentication with password authentication disabled
- Root SSH login disabled
- Tailscale-restricted remote administration
- Webmin access restricted to trusted LAN/Tailscale paths
- PhotoPrism multi-factor authentication
- Docker default seccomp filtering
- Docker `docker-default` AppArmor confinement for PhotoPrism
- Minimal host-port exposure for containerized services
- Separation of application, database, and administrative access paths
- Backup and configuration review procedures
- No credentials or private infrastructure data committed to this repository

Security remains an ongoing process. Future work includes stronger recovery testing, additional monitoring/alerting, access-policy refinement, and periodic review of the attack surface.

---

## Storage & Recovery

The current storage array uses two 1 TB hard drives in **RAID 0**. This provides capacity but **no redundancy**: failure of either member can make the array unusable.

> **RAID is storage configuration, not backup.**

The long-term storage plan is to move important data toward redundant storage while maintaining independent backups.

Automated PhotoPrism backups are already operational, but disaster-recovery work is intentionally still marked as in progress. Remaining work includes documented restore procedures, restore drills, failure scenarios, and stronger off-system/off-site recovery readiness.

---

## Monitoring

Server administration and monitoring use Webmin together with a custom project, **Webmin Homelab Monitor**.

The monitoring work covers areas such as:

- CPU and memory utilization
- RAID and storage health
- SMART information
- Docker/service state
- Network diagnostics
- Tailscale/Funnel state
- Firewall and SSH exposure
- Backup status
- Host health information

The monitor is kept modular rather than modifying Webmin core files, making it easier to maintain across Webmin upgrades.

See [Monitoring documentation](docs/monitoring.md).

---

## Documentation

Detailed documentation currently available:

- [Hardware](docs/hardware.md)
- [Operating System](docs/operating-system.md)
- [Docker](docs/docker.md)
- [Networking](docs/networking.md)
- [PhotoPrism](docs/photoprism.md)
- [Monitoring](docs/monitoring.md)

Additional architecture, security, backup/recovery, and troubleshooting documentation will be added as those areas are formalized.

---

## Repository Structure

This tree reflects the repository **as it exists now**, rather than presenting planned directories as already implemented.

```text
abrvn-homelab/
├── .gitignore
├── README.md
└── docs/
    ├── docker.md
    ├── hardware.md
    ├── monitoring.md
    ├── networking.md
    ├── operating-system.md
    └── photoprism.md
```

Future directories and documents will be added only when they contain useful, sanitized material.

---

## Lessons Learned

Several design lessons have shaped the project:

**Public exposure should be intentional.** ISP networking limitations and early remote-access experiments led to a design where administrative interfaces remain private and only the required application path is exposed.

**Containers do not need every service published to the host.** Internal services such as the database can communicate through Docker networking without unnecessary LAN or Internet exposure.

**RAID does not replace backups.** Storage availability, backup, and disaster recovery solve different problems and are treated separately.

**Security exceptions should be justified and tested.** The PhotoPrism deployment operated with broad seccomp/AppArmor exceptions until testing demonstrated that the application and Intel graphics device access worked under Docker's default protections.

**Customizations should survive upgrades.** The Webmin monitoring project is kept separate from Webmin core files so application updates do not overwrite project-specific functionality.

**Recovery matters as much as deployment.** A service is not considered fully resilient merely because backups exist; restoration procedures and failure drills are part of the roadmap.

---

## Project Roadmap

### Operational / established

- Linux host and core networking
- Docker / Docker Compose
- PhotoPrism + MariaDB
- Caddy reverse proxy
- Tailscale remote access and Funnel
- UFW and hardened SSH access
- PhotoPrism MFA
- Automated PhotoPrism backups
- Webmin administration
- Custom Webmin monitoring project

### In progress

- Repository documentation
- Backup validation and restore testing
- Disaster-recovery procedures
- Monitoring and alerting improvements
- Storage redundancy planning
- Additional sanitized architecture/configuration examples

---

## Security & Privacy Notice

This repository intentionally does **not** publish:

- Passwords or database credentials
- API keys or authentication tokens
- Private keys
- Personal photographs
- Private backups or databases
- Real private/internal IP addresses
- Tailscale addresses or private identifiers
- Personal account information
- Sensitive production paths or infrastructure identifiers

Examples should use placeholders or generalized values. Anyone adapting material from this repository should review it for their own environment rather than treating it as a drop-in production configuration.

---

## Why This Project Exists

ABRVN Homelab is both working infrastructure and a learning portfolio. It provides hands-on experience with Linux administration, Docker, networking, storage, secure remote access, backup/recovery planning, monitoring, troubleshooting, and infrastructure documentation.

The repository is intended to show not only **what** was deployed, but **why** design decisions were made and how the environment changed as problems were discovered and solved.
