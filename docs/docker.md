# Use of Docker

## Overview

Docker is the primary application deployment platform used in the ABRVN Homelab.

The project began as an effort to repurpose an older workstation into a practical self-hosted server. As additional services were introduced, Docker provided a way to deploy applications without turning the underlying Linux installation into a collection of tightly coupled application dependencies.

The current Docker environment primarily hosts:

* PhotoPrism
* MariaDB
* Caddy

Together, these containers form the core application stack for the homelab's self-hosted photo-management platform.

---

## Why Docker?

Docker is used because it provides a practical balance between **isolation, maintainability, reproducibility, portability, and learning**.

For this project, containers are not being used simply because they are a popular technology. They solve several real problems encountered while building and maintaining the homelab.

---

## Application Isolation

Each application runs within its own container environment instead of being installed directly onto the Linux host.

This separates application dependencies and configurations from the operating system.

For example, PhotoPrism and MariaDB have different software requirements, but they can operate as separate services while communicating through Docker networking.

This also makes troubleshooting more manageable because problems can often be isolated to a specific service instead of affecting the entire host.

A simplified model is:

```text
Linux Host
    │
    ▼
  Docker
    │
    ├── PhotoPrism
    │
    ├── MariaDB
    │
    └── Caddy
```

---

## Reproducibility

Docker Compose allows the application architecture to be described declaratively.

Instead of manually rebuilding every application installation, services, networks, volumes, and dependencies can be represented through Compose configuration.

This is particularly valuable for the homelab because the goal is not simply to have a server that works today.

The infrastructure should also be understandable and rebuildable after:

* Hardware replacement
* Operating-system migration
* Container failure
* Configuration mistakes
* Major upgrades
* Disaster-recovery events

This makes the Docker configuration an important part of the infrastructure documentation.

---

## Easier Maintenance

Containers allow individual services to be updated, restarted, or recreated independently.

For example, the reverse proxy can be restarted without necessarily requiring the database or PhotoPrism application to be restarted.

This reduces the impact of routine maintenance and makes experimentation safer.

It also encourages a modular approach to infrastructure:

> **A service should be replaceable without unnecessarily disrupting unrelated services.**

---

## Portability

The containerized architecture makes the application layer less dependent on the specific hardware used by the homelab.

If the server is eventually replaced or upgraded, the application stack can potentially be moved to another compatible Linux host without completely rebuilding every application installation from scratch.

This is one reason the project separates:

* Application software
* Container configuration
* Persistent application data
* Underlying hardware
* Storage infrastructure

The hardware may change over time while the application architecture remains largely consistent.

---

## Reduced Host Exposure

Docker also makes it possible for services to communicate internally without publishing every application port directly to the host.

This is important from a security perspective.

PhotoPrism and MariaDB do not need to be independently reachable from the LAN or Internet. Their communication can remain inside the Docker environment while Caddy provides the controlled application entry point.

The general model is:

```text
External Access
      │
      ▼
    Caddy
      │
      ▼
Docker Network
      │
      ├── PhotoPrism
      │
      └── MariaDB
```

Reducing unnecessary host port publishing helps minimize the number of services directly exposed to the host network.

---

# Current Architecture

The current application stack consists of three primary containers:

```text
                         Internet
                            │
                            ▼
                    Remote Access Layer
                            │
                            ▼
                     Host Loopback
                            │
                            ▼
                     ┌───────────┐
                     │   Caddy   │
                     │  Reverse  │
                     │   Proxy   │
                     └─────┬─────┘
                           │
                    Docker Network
                           │
                           ▼
                    ┌─────────────┐
                    │  PhotoPrism │
                    └──────┬──────┘
                           │
                    Docker Network
                           │
                           ▼
                    ┌─────────────┐
                    │   MariaDB   │
                    └─────────────┘
```

The architecture intentionally separates the host-facing network layer from internal application communication.

Caddy provides the controlled entry point while PhotoPrism and MariaDB remain inside the Docker environment.

---

# PhotoPrism

PhotoPrism is the primary application running in the Docker environment.

It provides the self-hosted interface for managing and browsing the homelab's photo and video library.

The PhotoPrism container is intentionally not published directly to the host network.

Instead, requests are routed through Caddy.

This creates a single controlled application entry point rather than independently exposing the application's internal port.

---

# MariaDB

MariaDB provides the database backend used by PhotoPrism.

The database is separated into its own container rather than being installed directly onto the Linux host.

PhotoPrism communicates with MariaDB through Docker's internal networking.

MariaDB does not need to be directly accessible from the LAN or Internet for normal PhotoPrism operation.

This reduces unnecessary network exposure and keeps database access limited to the services that actually require it.

---

# Caddy

Caddy serves as the reverse proxy for the PhotoPrism application.

Its responsibilities include:

* Receiving incoming web requests
* Forwarding requests to PhotoPrism
* Providing a controlled application entry point
* Integrating with the homelab's remote-access architecture
* Handling the web-facing portion of the application stack

Caddy is the only component in the application stack that requires a host-facing HTTP entry point.

The architecture therefore resembles:

```text
Remote Client
      │
      ▼
Remote Access Layer
      │
      ▼
    Caddy
      │
      ▼
 PhotoPrism
      │
      ▼
  MariaDB
```

---

# Docker Networking

Caddy and PhotoPrism communicate through a dedicated Docker network.

This allows containers to communicate using Docker's internal networking instead of requiring every service to expose its application port to the host.

The architecture can therefore be viewed as two separate networking layers:

```text
External / Host Network
        │
        ▼
 Remote Access Layer
        │
        ▼
      Caddy
        │
        ▼
────────────────────────
      Docker Network
────────────────────────
        │
        ├── PhotoPrism
        │
        └── MariaDB
```

This separation is intentional.

The host handles external connectivity and security controls, while the containers handle application-to-application communication.

This design also makes it easier to add future applications without automatically exposing their internal ports to the LAN.

---

# Storage

PhotoPrism's original media library is stored outside the container.

The actual storage location is intentionally omitted from this public documentation.

Instead, the architecture can be represented as:

```text
PhotoPrism
    │
    ▼
Container Mount
    │
    ▼
Linux Filesystem
    │
    ▼
Storage Array
    │
    ▼
Physical Drives
```

Keeping the photo library outside the container provides an important separation between **application software** and **persistent user data**.

The PhotoPrism container can therefore be replaced without inherently replacing the underlying photo library.

This distinction is important when designing a recoverable self-hosted application.

---

# Persistent Data

Container instances themselves should generally be considered replaceable.

Persistent data must be treated differently.

The homelab separates several categories of data:

* Container images
* Container configuration
* Application databases
* Photo and video originals
* Generated application data
* Backup data

This separation allows the application layer to be rebuilt while preserving important persistent data through the separate backup and recovery strategy.

The underlying principle is:

> **Containers are replaceable; important data is not.**

---

# Container Lifecycle

The general service lifecycle is:

```text
Docker Compose Configuration
            │
            ▼
        Pull Image
            │
            ▼
       Create Container
            │
            ▼
       Attach Volumes
            │
            ▼
       Attach Networks
            │
            ▼
        Start Service
            │
            ▼
       Health Checks
            │
            ▼
      Normal Operation
```

This model has also become useful as a learning tool.

Rather than treating Docker as a black box, the homelab is used to understand how:

* Images
* Containers
* Networks
* Volumes
* Dependencies
* Host resources
* Application configuration

interact with one another.

---

# Configuration Management

Docker Compose configurations are treated as infrastructure rather than disposable setup files.

Production configuration can contain sensitive information such as:

* Database credentials
* Application secrets
* Authentication information
* Private network details
* TLS-related configuration

These values should **never** be committed to the public GitHub repository.

The repository should instead contain sanitized examples where useful.

For example:

```text
infrastructure/
└── photoprism/
    ├── compose.example.yml
    └── README.md
```

Example configurations should use placeholders rather than real passwords, tokens, private keys, personal paths, or other sensitive information.

---

# Security Considerations

Docker is treated as an **application deployment and isolation mechanism**, not as a complete security boundary.

The homelab uses additional controls, including:

* UFW host firewall
* Tailscale remote connectivity
* Restricted administrative services
* Minimal host port publishing
* Docker network isolation
* Reverse-proxy architecture
* Application authentication
* Regular software updates
* Backup and recovery procedures

The goal is to reduce unnecessary exposure while maintaining reliable service access.

A container being isolated does not automatically make the application secure. The host operating system, network, application configuration, credentials, and exposed services must all be considered.

---

# Current Container State

| Container  | Purpose                      | Host Port Exposure |
| ---------- | ---------------------------- | ------------------ |
| PhotoPrism | Photo management application | None               |
| MariaDB    | PhotoPrism database          | None               |
| Caddy      | Reverse proxy                | Loopback HTTP      |

PhotoPrism and MariaDB expose their application ports within the Docker environment rather than directly publishing them to the host network.

Caddy provides the controlled host-facing entry point.

This arrangement follows the principle of exposing only what is actually required.

---

# Backup and Recovery

Docker itself is not considered a backup system.

The homelab's separate backup strategy protects important persistent data and configuration.

A recovery process should be capable of:

1. Reinstalling the operating system if necessary
2. Installing Docker
3. Restoring the Compose configuration
4. Restoring required environment variables and secrets securely
5. Restoring persistent application data
6. Starting the container stack
7. Verifying application functionality

The goal is to make the environment **rebuildable** rather than dependent on a particular container instance.

This distinction becomes especially important when containers are intentionally treated as disposable infrastructure.

---

# Troubleshooting and Lessons Learned

Building the PhotoPrism environment made Docker a practical systems-administration learning exercise.

One of the most important lessons has been that containerization does not eliminate the need to understand the underlying operating system.

The containers still depend on:

* Host storage
* File permissions
* Networking
* DNS
* Firewall rules
* CPU and memory resources
* Kernel functionality
* Docker configuration
* Backup and recovery procedures

A problem that appears to be a Docker problem may actually originate from the host.

For example:

```text
Host Storage
      │
      ▼
Filesystem Permissions
      │
      ▼
Container Mount
      │
      ▼
Application Access
```

A failure at any point can appear as an application-level problem.

---

# A Practical Docker Lesson

One of the more useful lessons from building the homelab was that seemingly small configuration problems can prevent an otherwise correct Docker deployment from starting or behaving as expected.

This reinforced the importance of validating configuration rather than assuming that a Compose file is correct simply because its structure appears reasonable.

The troubleshooting process therefore emphasizes:

* Reading container logs
* Inspecting container state
* Validating Compose configuration
* Checking network connectivity
* Verifying volume mounts
* Checking filesystem permissions
* Confirming service dependencies
* Testing changes incrementally

This approach is more valuable than repeatedly restarting containers without understanding the underlying failure.

---

# Docker and Infrastructure Design

Docker is only one layer of the homelab.

The broader infrastructure can be represented as:

```text
Physical Hardware
       │
       ▼
   Linux Host
       │
       ├── Storage
       ├── Networking
       ├── Firewall
       └── Administration
              │
              ▼
            Docker
              │
       ┌──────┼──────┐
       ▼      ▼      ▼
    Caddy  PhotoPrism  MariaDB
       │      │
       └──────┴──────────┐
                         ▼
                   Persistent Data
                         │
                         ▼
                    Backup System
```

This layered architecture helps isolate responsibilities.

The Linux host manages hardware and system resources.

Docker manages application deployment.

The containers provide application functionality.

The storage and backup layers protect persistent information.

---

# Future Improvements

Planned Docker improvements include:

* Sanitized Docker Compose examples for GitHub
* Container health monitoring
* Automated service-status reporting
* Resource monitoring
* Documented container-update procedures
* Improved backup verification
* Documented disaster-recovery procedures
* Additional Docker security hardening
* Better separation of production and development configurations
* Expansion to additional self-hosted services
* Integration with the custom Webmin dashboard

The dashboard will eventually provide a higher-level view of container health alongside CPU, memory, storage, network, and system information.

---

# Design Philosophy

The Docker architecture follows a simple principle:

> **Treat containers as replaceable application infrastructure and persistent data as something that must be deliberately protected.**

The purpose of Docker in this homelab extends beyond simply running PhotoPrism.

It provides a practical environment for learning:

* Containerization
* Linux administration
* Networking
* Application deployment
* Service isolation
* Infrastructure reproducibility
* Troubleshooting
* Security
* Disaster recovery

The broader goal of the ABRVN Homelab is not simply to run services.

It is to **build, document, troubleshoot, secure, recover, and continuously improve a real self-hosted environment** while developing practical infrastructure and cybersecurity skills.
