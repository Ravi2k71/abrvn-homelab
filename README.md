# ABRVN Homelab

A hands-on self-hosted Linux infrastructure project focused on systems administration, cybersecurity, networking, storage, monitoring, automation, and disaster recovery.

The homelab is designed to provide private self-hosted services for personal and family use while serving as a practical environment for developing and documenting real-world IT and cybersecurity skills.

---

## Project Overview

The server currently hosts a self-hosted photo management platform using **PhotoPrism**, supported by a Docker-based application stack, RAID storage, reverse proxying, Tailscale networking, firewall controls, Samba file sharing, Webmin administration, and automated backups.

The project is continuously evolving, with plans to expand the server into a centralized monitoring and management platform through a custom Webmin-based dashboard.

The goal is not simply to run applications, but to design, secure, monitor, maintain, and document a complete infrastructure environment.

---

## Current Infrastructure

| Component           | Technology          |
| ------------------- | ------------------- |
| Operating System    | Pop!_OS 24.04 LTS   |
| CPU                 | Intel Core i5-6500  |
| Memory              | 16 GB RAM           |
| Storage             | 2 × 1 TB HDD RAID 0 |
| Photo Management    | PhotoPrism CE       |
| Database            | MariaDB             |
| Containers          | Docker              |
| Reverse Proxy       | Caddy               |
| Remote Access       | Tailscale           |
| Public Photo Access | Tailscale Funnel    |
| File Sharing        | Samba               |
| Firewall            | UFW                 |
| Server Management   | Webmin              |
| Automation          | Bash / systemd      |
| Backup Strategy     | 3-2-1               |
| Version Control     | Git / GitHub        |

---

## Architecture

The infrastructure separates applications, storage, administration, and networking while minimizing unnecessary network exposure.

```text
                              Internet
                                  │
                                  │
                         Tailscale Funnel
                                  │
                                  ▼
                              Caddy
                                  │
                                  ▼
                           PhotoPrism CE
                                  │
                                  ▼
                              MariaDB
                                  │
                                  ▼
                           RAID Storage


       ┌─────────────────────────────────────────────┐
       │                Linux Server                 │
       │                                             │
       │  PhotoPrism       Docker       Caddy        │
       │  MariaDB          Samba        Webmin       │
       │  Tailscale        UFW          Monitoring   │
       │                                             │
       └─────────────────────────────────────────────┘
                    │                    │
                    ▼                    ▼
             Local Network           Tailscale
                    │                    │
                    ▼                    ▼
             Computers/Devices      Remote Access
```

Administrative services are intentionally kept separate from the publicly accessible PhotoPrism path.

Detailed architecture documentation will be maintained in:

`docs/architecture.md`

---

## Security

Security is treated as a core part of the project rather than an afterthought.

Current security controls include:

* UFW host-based firewall
* Restricted administrative access
* Tailscale-based networking
* TOTP multi-factor authentication for supported applications/accounts
* Docker services isolated from unnecessary host port exposure
* Reverse proxy architecture through Caddy
* Restricted Webmin network access
* Automated configuration and database backups
* 3-2-1 backup strategy
* Encrypted offline backup storage
* Regular system maintenance and recovery planning

Administrative services are intentionally not exposed directly to the public Internet.

Future security improvements will include additional access-control hardening, security reviews, network segmentation, and expanded monitoring.

---

## Storage & RAID

The current server uses two 1 TB hard drives configured as a **RAID 0 array**.

RAID 0 provides increased usable capacity and performance but does **not** provide redundancy. The array is therefore treated as production storage rather than a backup mechanism.

The homelab's data protection strategy relies on the separate 3-2-1 backup system.

Future storage plans include evaluating larger drives and transitioning primary storage to a redundant configuration such as RAID 1.

---

## Backup & Recovery

The server uses an automated backup system designed to protect application data and infrastructure configuration.

The backup process includes:

* PhotoPrism database dumps
* PhotoPrism configuration
* Docker Compose configuration
* Caddy configuration
* Webmin configuration
* Tailscale-related automation
* Backup scripts
* Backup manifest
* Database archive verification
* Retention management

Backups are additionally protected through encrypted offline storage and secondary copies.

The backup strategy follows the general **3-2-1 principle**:

* Multiple copies of important data
* Multiple storage locations/media
* At least one offline or otherwise isolated copy

Future disaster-recovery work will include regular restoration testing and documented procedures for recovering from complete server failure.

Detailed procedures will be documented in:

* `docs/backups.md`
* `docs/disaster-recovery.md`

---

## Monitoring & Automation

A major component of the project is the development of a custom server monitoring dashboard built around Webmin.

The planned dashboard will provide a centralized view of:

* CPU utilization
* Memory utilization
* Intel GPU activity
* RAID health
* Storage capacity
* SMART health
* Disk temperatures
* Docker containers
* PhotoPrism
* MariaDB
* Caddy
* Tailscale connectivity
* Firewall/security status
* Backup status
* Overall system health

### Automated Reporting

The monitoring system is also planned to provide automated email-based reporting and alerting.

Planned functionality includes:

* Periodic server health reports
* Backup success/failure notifications
* Storage and RAID warnings
* Disk-health alerts
* Service failure alerts
* Security-related notifications
* System health summaries

The objective is to move beyond simply checking the server manually and toward proactive infrastructure monitoring.

---

## Custom Server Dashboard

The planned **ABRVN Server Dashboard** will extend Webmin with custom monitoring functionality rather than replacing Webmin entirely.

The project will use:

* Custom monitoring collectors
* Webmin module development
* Webmin theme customization where appropriate
* Bash/system utilities
* systemd automation
* Service health checks
* External API integrations
* Weather information
* Automated reporting

The dashboard is intended to provide an interface tailored specifically to the infrastructure running on this server.

Rather than modifying Webmin's core files whenever possible, custom functionality will be separated into independently maintained modules, collectors, and configuration so that Webmin upgrades can be performed more safely.

---

## Documentation

This repository documents not only the final configuration, but also the reasoning behind architectural, operational, and security decisions.

Planned documentation includes:

* `docs/architecture.md` — Overall system architecture
* `docs/hardware.md` — Server hardware and storage
* `docs/operating-system.md` — Operating system configuration
* `docs/docker.md` — Container architecture
* `docs/photoprism.md` — PhotoPrism deployment
* `docs/networking.md` — Network architecture
* `docs/tailscale.md` — Remote-access architecture
* `docs/security.md` — Security controls and hardening
* `docs/backups.md` — Backup strategy
* `docs/monitoring.md` — Monitoring and alerting
* `docs/disaster-recovery.md` — Recovery procedures
* `docs/troubleshooting.md` — Problems encountered and solutions

The documentation is intended to explain **why** particular technologies and configurations were selected, not simply provide a collection of commands.

---

## Troubleshooting & Lessons Learned

Real infrastructure problems are documented as part of the project.

This includes:

* Operating-system upgrades and recovery
* Package and repository conflicts
* Networking problems
* Remote-access issues
* Docker configuration problems
* Service recovery
* Storage and RAID troubleshooting
* Firewall configuration
* Remote administration
* Security decisions
* Architectural changes

Major incidents will be documented as case studies describing:

1. The original problem
2. The symptoms observed
3. The investigation process
4. The root cause
5. The solution
6. Security or reliability implications
7. Lessons learned
8. Preventative measures

The purpose is to demonstrate troubleshooting and problem-solving rather than simply documenting successful commands.

---

# Proposed Future Plans

The homelab is an ongoing project, with future improvements planned across infrastructure, security, monitoring, automation, and reliability.

## Infrastructure

* Expand storage capacity as the photo library grows.
* Transition the primary storage array from RAID 0 to a redundant configuration such as RAID 1.
* Evaluate larger HDDs and dedicated backup storage.
* Improve hardware monitoring and health reporting.
* Evaluate hardware upgrades where they provide meaningful improvements in reliability or performance.

## Monitoring & Dashboard

* Complete the **ABRVN Server Dashboard** as a custom Webmin module.
* Add real-time CPU, memory, GPU, storage, RAID, and service monitoring.
* Add SMART health monitoring and disk-temperature reporting.
* Monitor Docker containers and individual application health.
* Add PhotoPrism, MariaDB, Caddy, Samba, and Tailscale status monitoring.
* Develop historical system metrics and trend reporting.
* Implement automated email health reports.
* Implement alerts for critical failures.
* Add weather information as a dashboard integration.
* Create a centralized infrastructure-health overview.

## Security

* Further harden the server and administrative services.
* Implement stronger administrative access controls.
* Expand Tailscale-based remote administration while keeping administrative services inaccessible from the public Internet.
* Implement and maintain Webmin TOTP multi-factor authentication.
* Review firewall rules periodically and remove unnecessary access.
* Develop and maintain a documented threat model.
* Perform periodic security reviews of exposed services and network paths.
* Evaluate additional network segmentation and isolation techniques.
* Document security decisions and architectural tradeoffs.

## Automation

* Expand systemd-based automation for maintenance and monitoring.
* Automate additional health checks.
* Improve automated backup verification.
* Add automated notifications for backup failures and infrastructure problems.
* Automate routine maintenance reporting.
* Develop reusable Bash utilities for server administration.
* Create automated infrastructure-health checks.

## Backup & Disaster Recovery

* Continue expanding the existing 3-2-1 backup strategy.
* Add additional backup destinations as storage requirements increase.
* Perform scheduled restoration tests rather than only verifying that backups exist.
* Document complete disaster-recovery procedures.
* Develop recovery procedures for:

  * Storage failure
  * Operating-system failure
  * Docker/application failure
  * Configuration loss
  * Database corruption
  * Complete server replacement
* Maintain recovery documentation that can be followed without relying on undocumented knowledge.

## Services & Applications

Potential future self-hosted services may include:

* Additional family-focused applications
* Network monitoring
* Home automation services
* Additional media-management applications
* Private file-management services
* Internal documentation or knowledge-management systems
* Additional Docker-based applications

New services will be evaluated based on:

* Usefulness
* Security requirements
* Resource consumption
* Maintenance requirements
* Backup requirements
* Learning value

The goal is not to run as many services as possible, but to build a useful and maintainable infrastructure environment.

## Networking

* Further improve Tailscale networking and access controls.
* Document the interaction between the local LAN, Docker networks, Tailscale, Caddy, and externally accessible services.
* Improve network monitoring and connectivity reporting.
* Evaluate additional network segmentation.
* Continue minimizing publicly exposed services.
* Document remote-access architecture and trust boundaries.

## Portfolio & Documentation

* Expand the repository into a complete technical portfolio documenting the evolution of the homelab.
* Add architecture diagrams and network diagrams.
* Document major troubleshooting incidents and their resolutions.
* Document security decisions and tradeoffs.
* Add screenshots of the custom monitoring dashboard.
* Maintain configuration examples that can be reproduced without exposing private information.
* Document lessons learned from operating and maintaining the infrastructure over time.
* Create technical case studies demonstrating troubleshooting and security analysis.

> Future plans are intentionally flexible. Features will be implemented based on reliability, security, learning value, and actual usefulness to the homelab.

---

# Project Goals

The long-term goals of this homelab are to:

1. Maintain a reliable self-hosted family photo platform.
2. Improve Linux system administration skills.
3. Develop practical networking and cybersecurity experience.
4. Build automated monitoring and alerting.
5. Implement reliable backup and disaster-recovery procedures.
6. Develop a custom server management dashboard.
7. Document infrastructure decisions and troubleshooting experiences.
8. Practice secure system design and operational security.
9. Develop automation and infrastructure-management skills.
10. Create a practical portfolio demonstrating hands-on IT experience.

---

# Technologies

**Linux · Pop!_OS · Docker · PhotoPrism · MariaDB · Caddy · Tailscale · Webmin · Samba · UFW · RAID · Bash · systemd · Git · GitHub**

---

# Repository Structure

The repository is planned to follow a modular structure separating documentation, infrastructure configuration, monitoring, automation, and security material.

```text
abrvn-homelab/
├── README.md
│
├── docs/
│   ├── architecture.md
│   ├── hardware.md
│   ├── operating-system.md
│   ├── photoprism.md
│   ├── docker.md
│   ├── networking.md
│   ├── tailscale.md
│   ├── security.md
│   ├── backups.md
│   ├── monitoring.md
│   ├── disaster-recovery.md
│   └── troubleshooting.md
│
├── dashboard/
│   ├── README.md
│   ├── webmin-module/
│   ├── collectors/
│   │   ├── cpu/
│   │   ├── memory/
│   │   ├── gpu/
│   │   ├── raid/
│   │   ├── storage/
│   │   ├── docker/
│   │   ├── backup/
│   │   └── security/
│   ├── integrations/
│   │   ├── photoprism/
│   │   ├── tailscale/
│   │   └── weather/
│   └── systemd/
│
├── infrastructure/
│   ├── photoprism/
│   │   ├── compose.example.yml
│   │   └── README.md
│   ├── caddy/
│   │   ├── Caddyfile.example
│   │   └── README.md
│   └── samba/
│
├── scripts/
│   ├── photoprism-backup.sh
│   └── ...
│
├── security/
│   ├── firewall.md
│   ├── authentication.md
│   └── threat-model.md
│
├── diagrams/
│   └── architecture.png
│
└── screenshots/
```

---

## Security & Privacy Notice

This repository intentionally does **not** contain:

* Passwords
* API keys
* Authentication tokens
* Tailscale authentication keys
* Private SSH keys
* TLS private keys
* Database credentials
* `.env` files containing secrets
* Personal/family photos
* Backup archives
* VeraCrypt containers
* Sensitive network information

Configuration examples should use placeholders where sensitive information would otherwise be required.

---

## Status

🚧 **Active Development**

This project is continuously evolving.

Current development priorities include:

1. Maintaining PhotoPrism reliability
2. Maintaining backup and recovery readiness
3. Improving Webmin administration
4. Restoring and expanding GPU monitoring
5. Developing the ABRVN Server Dashboard
6. Expanding infrastructure monitoring
7. Improving security hardening
8. Expanding documentation
9. Building automated health reporting
10. Continuing to develop the homelab as a cybersecurity and IT portfolio project
