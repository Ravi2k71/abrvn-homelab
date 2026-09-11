# Operating System

## Overview

The ABRVN Homelab server currently runs **Pop!_OS 24.04 LTS** as its operating system.

Pop!_OS provides the Linux foundation for the homelab's containerized applications, storage services, networking tools, system administration, and monitoring infrastructure.

## System Configuration

| Component               | Configuration     |
| ----------------------- | ----------------- |
| Operating System        | Pop!_OS 24.04 LTS |
| Desktop Environment     | GNOME             |
| Display Server          | X11               |
| Architecture            | 64-bit            |
| Primary Management Tool | Webmin            |
| Container Platform      | Docker            |

## Role in the Homelab

The operating system provides the underlying platform for the server's infrastructure and services, including:

* Docker and containerized applications
* PhotoPrism
* MariaDB
* Caddy
* Samba file sharing
* RAID storage management
* Tailscale networking
* Webmin system administration
* UFW firewall management
* Hardware and system monitoring
* Backup and recovery processes
* Custom monitoring and dashboard development

## System Administration

**Webmin** is used as the primary graphical administration interface for the server.

Webmin provides centralized management for many aspects of the Linux system while allowing command-line tools to be used when more direct or specialized configuration is required.

The project intentionally uses both graphical and command-line administration. This provides flexibility while also allowing the underlying Linux configuration and troubleshooting processes to be understood rather than hidden behind a management interface.

## Software Management

System software and security updates are managed through the operating system's package management system.

Docker is maintained separately through its official package repository and is used to isolate application services from the host operating system.

Third-party repositories are kept to a minimum and are documented when required by the homelab.

## Reliability Considerations

The server is treated as infrastructure rather than simply a desktop computer. Configuration changes are therefore evaluated based on their potential effect on:

* Data availability
* Containerized services
* Network connectivity
* Remote administration
* Storage access
* Backup operations
* System recovery

Changes that could affect the server's ability to boot or provide core services are tested carefully before being applied to the production configuration.

## Future Improvements

Planned operating-system and platform improvements include:

* Continued system and security updates
* Improved automated monitoring
* Hardware health monitoring
* More comprehensive service health checks
* Automated reporting and alerting
* Improved disaster-recovery documentation
* Configuration backup and version control
* Additional security hardening
* Evaluation of virtualization and additional self-hosted workloads
