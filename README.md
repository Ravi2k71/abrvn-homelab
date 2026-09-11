# ABRVN Homelab — Self-Hosted Infrastructure & Security Lab

A self-hosted infrastructure lab focused on Linux administration, containerization, storage, networking, remote access, security hardening, monitoring, automation, and disaster recovery.

The project started as a way to build a practical self-hosted photo server and has grown into a broader environment for learning infrastructure and security engineering through hands-on experimentation.

> **Status:** Active development

---

## Overview

The ABRVN Homelab is a personal infrastructure environment built around a repurposed Dell Precision workstation.

The primary application is **PhotoPrism**, which provides a self-hosted photo management platform. Supporting infrastructure includes Docker, MariaDB, Caddy, Tailscale, RAID storage, Samba, UFW, Webmin, automated backups, and custom monitoring.

The project is designed around two goals:

1. Build useful self-hosted services for everyday use.
2. Use the infrastructure as a practical learning environment for systems administration and cybersecurity.

Rather than treating the server as a simple application host, the project documents the architecture, security decisions, failures, recovery procedures, and lessons learned along the way.

---

## Current Infrastructure

| Component          | Technology                 |
| ------------------ | -------------------------- |
| Server             | Dell Precision 3620        |
| CPU                | Intel Core i5-6500         |
| Memory             | 16 GB RAM                  |
| GPU                | Intel integrated graphics  |
| Operating System   | Pop!_OS 24.04 LTS          |
| Container Platform | Docker / Docker Compose    |
| Photo Management   | PhotoPrism CE              |
| Database           | MariaDB                    |
| Reverse Proxy      | Caddy                      |
| Remote Access      | Tailscale                  |
| File Sharing       | Samba                      |
| Firewall           | UFW                        |
| Administration     | Webmin                     |
| Storage            | 2 × 1 TB HDD               |
| RAID               | RAID 0                     |
| Backup Strategy    | 3-2-1 backup approach      |
| Monitoring         | Webmin + custom collectors |
| Repository         | GitHub                     |

---

## Architecture

The infrastructure is intentionally separated into application, networking, storage, and administrative layers.

```text
                         Remote Access
                              │
                              ▼
                       Tailscale / Funnel
                              │
                              ▼
                         Caddy Proxy
                              │
                              ▼
                         PhotoPrism
                              │
                    ┌─────────┴─────────┐
                    ▼                   ▼
              Photo Storage          MariaDB
                    │
                    ▼
                 RAID Storage
                    │
                    ▼
             Backup Infrastructure
```

Administrative services such as SSH, Webmin, Samba, and server management are kept separate from the public-facing application path wherever practical.

---

## PhotoPrism

PhotoPrism is the primary application running on the server.

The service provides:

* Self-hosted photo management
* Automatic indexing and organization
* Photo and video browsing
* Metadata management
* User authentication
* Database-backed application storage
* Remote access

PhotoPrism runs as a Docker container and communicates with a separate MariaDB container over an internal Docker network.

Photo storage is provided through a host-mounted storage volume rather than being stored inside the application container.

For public documentation, actual filesystem locations and private infrastructure identifiers are intentionally omitted.

---

## Docker

Docker is used to isolate the primary application services from the host operating system.

Current containerized services include:

* PhotoPrism
* MariaDB
* Caddy

The containers communicate through dedicated Docker networks.

The architecture avoids unnecessary host port exposure. Services that do not need direct access from the host network remain accessible only through their Docker networks.

This provides a cleaner separation between:

* Public-facing services
* Internal application services
* Database services
* Host administration

Docker Compose is used to define and manage the application stack.

---

## Networking & Remote Access

Remote access is provided through **Tailscale**.

The project originally explored traditional direct port forwarding and public reverse-proxy access, but the ISP network environment presented limitations that made this approach impractical.

This led to using a mesh-VPN-based architecture instead.

The current design uses:

* Tailscale for secure remote administration
* Tailscale Funnel for public PhotoPrism access
* Caddy as the reverse-proxy layer
* UFW for host-level firewall restrictions
* Docker networks for internal service isolation

A major design principle is **least necessary exposure**.

Services are not exposed publicly simply because they can be. Public access is limited to the application that requires it, while administrative interfaces remain restricted.

---

## Security

Security is a core consideration of the homelab, with an emphasis on minimizing attack surface, restricting administrative access, and separating publicly accessible services from internal infrastructure.

Current security practices include:

* UFW host-based firewall with service-specific network restrictions
* Tailscale for secure remote connectivity
* Restricted administrative access to services such as SSH and Webmin
* Docker services configured without unnecessary host port exposure
* Caddy used as a controlled reverse-proxy layer
* Separation of application services from administrative services
* Automated backups of important application and configuration data
* Encrypted offline backup storage
* Regular system updates and configuration review
* Avoiding direct public exposure of administrative services

The server follows a principle of **least necessary exposure**: services are only made accessible where there is a specific requirement, and administrative interfaces are kept separate from the public PhotoPrism access path.

Security improvements are treated as an ongoing process. Future work will include additional access controls, multi-factor authentication, Tailscale access policies, network segmentation, security monitoring, threat-model documentation, and periodic security reviews.

---

## Storage & RAID

The server currently uses two 1 TB hard drives configured as **RAID 0**.

RAID 0 provides increased usable capacity and can improve sequential storage performance, but it provides **no redundancy**.

A failed drive can result in the loss of the entire RAID array.

For that reason:

> **RAID is treated as a storage configuration, not a backup.**

The homelab uses a separate 3-2-1 backup strategy to protect important data.

Future storage expansion is expected to focus on redundant storage configurations such as RAID 1 rather than relying on RAID 0 for important data.

---

## Backup & Recovery

Backups are designed around the 3-2-1 principle:

* Multiple copies of important data
* Copies stored on different types of storage
* At least one copy maintained offline

Important backup data is encrypted before being stored offline.

The backup strategy is intended to protect against:

* Hardware failure
* Accidental deletion
* File corruption
* Configuration mistakes
* System failure
* RAID failure
* Malware or other destructive events

Recovery procedures will be documented separately as the project develops.

---

## Monitoring & Automation

The homelab includes a custom monitoring effort built around Webmin.

The planned dashboard provides a centralized view of server health, including:

* CPU utilization
* Memory usage
* GPU activity
* RAID health
* Storage utilization
* Disk health / SMART information
* Docker container status
* PhotoPrism status
* MariaDB status
* Caddy status
* Network connectivity
* Tailscale status
* Firewall/security status
* Backup status
* System notifications
* Weather information

The monitoring system is being developed as a modular project rather than modifying Webmin's core source files directly.

Custom collectors and integrations are intended to remain separate from the base Webmin installation so that Webmin updates do not unnecessarily overwrite project-specific changes.

---

## Troubleshooting & Lessons Learned

A major purpose of this project is documenting failures rather than only documenting successful configurations.

Some of the lessons from developing the homelab include:

### Don't assume a service should be publicly exposed

Early experimentation with traditional public access demonstrated how ISP networking limitations can affect infrastructure design.

The final architecture uses Tailscale to avoid depending on unrestricted inbound connectivity.

### Containers should not automatically expose every service

The database does not need to be directly accessible from the LAN or Internet.

Keeping MariaDB on an internal Docker network reduces unnecessary exposure.

### RAID does not replace backups

A RAID array can improve availability or storage performance, but it cannot protect against every form of data loss.

The backup strategy therefore exists independently of the RAID configuration.

### Configuration should be documented before making major changes

Several infrastructure issues reinforced the importance of recording configuration decisions, dependencies, and recovery procedures before performing upgrades or restructuring services.

### Customizations should survive upgrades

Directly modifying files belonging to a package or application can create maintenance problems when that software is upgraded.

The dashboard project therefore aims to keep custom functionality modular and separated from Webmin's core installation.

---

## Project Goals

The long-term goals of the homelab include:

* Build a reliable self-hosted photo platform
* Develop practical Linux administration skills
* Learn Docker and container networking
* Practice secure remote-access design
* Develop storage and backup strategies
* Build monitoring and automation systems
* Learn system recovery and disaster-recovery techniques
* Practice security hardening
* Create reusable infrastructure documentation
* Develop a custom Webmin dashboard
* Turn the homelab into a practical cybersecurity and infrastructure portfolio project

---

## Technologies

### Operating System

* Pop!_OS
* Linux
* systemd
* UFW

### Infrastructure

* Docker
* Docker Compose
* Webmin
* Samba
* RAID / mdadm

### Applications

* PhotoPrism
* MariaDB
* Caddy

### Networking

* Tailscale
* Reverse proxy
* Docker networking

### Security

* UFW
* Tailscale access controls
* Authentication
* Multi-factor authentication
* Encrypted backups
* Least-privilege / least-exposure principles

### Development & Documentation

* Git
* GitHub
* Bash
* Markdown

---

## Repository Structure

```text
abrvn-homelab/
├── README.md
├── .gitignore
├── docs/
│   ├── architecture.md
│   ├── hardware.md
│   ├── operating-system.md
│   ├── photoprism.md
│   ├── docker.md
│   ├── networking.md
│   ├── backups.md
│   ├── monitoring.md
│   ├── disaster-recovery.md
│   └── troubleshooting.md
├── dashboard/
│   ├── README.md
│   ├── webmin-module/
│   ├── collectors/
│   ├── integrations/
│   └── systemd/
├── infrastructure/
│   ├── photoprism/
│   ├── caddy/
│   └── samba/
├── scripts/
├── security/
├── diagrams/
└── screenshots/
```

Only components that are actually implemented will be added to the repository. Planned directories and features are documented as future work rather than represented as completed infrastructure.

---

## Security & Privacy Notice

This repository intentionally does **not** contain:

* Passwords
* API keys
* Private keys
* Authentication tokens
* Personal photographs
* Private backups
* Database credentials
* Real internal IP addresses
* Tailscale IP addresses or identifiers
* Private network configuration
* Real filesystem locations
* Personal account information
* Sensitive infrastructure identifiers

Examples in the documentation use sanitized paths, placeholders, or generalized architecture diagrams where appropriate.

The repository is intended to demonstrate infrastructure design and engineering decisions without exposing private information from the underlying homelab.

---

## Project Status

The homelab is an ongoing project.

Current priorities include:

1. Maintain a stable PhotoPrism deployment
2. Continue improving backup and recovery readiness
3. Document the existing infrastructure
4. Develop the custom Webmin dashboard
5. Add monitoring and automation
6. Improve security controls
7. Document disaster recovery procedures
8. Continue testing and refining the infrastructure

The project will evolve as new services, hardware, security controls, and automation are added.
