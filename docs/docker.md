# Docker

## Overview

Docker is the primary application deployment platform used in the ABRVN Homelab.

This homelab started as an effort to repurpose a **Dell Precision 3620 workstation** into a practical self-hosted server. As more services were added, Docker provided a way to experiment with different applications without turning the underlying Pop!_OS installation into a collection of tightly coupled application dependencies.

The current Docker environment primarily hosts:

* PhotoPrism
* MariaDB
* Caddy

These services work together to provide the homelab's self-hosted photo management platform.

## Why Docker?

Docker is used because it provides a practical balance between **learning, isolation, maintainability, and reproducibility**.

For this project, containers are not being used simply because they are a popular technology. They solve several real problems encountered while building and maintaining the homelab.

### Application Isolation

Each application runs within its own container environment instead of being installed directly onto the Pop!_OS host.

This keeps application dependencies and configurations separated from the operating system.

For example, PhotoPrism and MariaDB have different software requirements, but they can operate independently while communicating through Docker networking.

This also makes troubleshooting more straightforward because problems can often be isolated to a particular service rather than affecting the entire host.

### Reproducibility

Docker Compose allows the application architecture to be described declaratively.

Instead of manually rebuilding every application installation, the services, networks, volumes, and dependencies can be defined in a Compose configuration.

This is particularly important for this homelab because the goal is not just to have a server that works today, but to understand how the infrastructure could be rebuilt after a failure or migration.

### Easier Maintenance

Containers allow individual services to be updated, restarted, or recreated independently.

For example, Caddy can be restarted without necessarily restarting PhotoPrism or MariaDB.

This reduces the impact of maintenance and makes experimentation safer.

### Portability

The containerized architecture makes the application stack less dependent on the specific Dell Precision 3620 hardware.

If the server is eventually replaced or upgraded, the application layer can potentially be moved to another Linux host without completely rebuilding the environment from scratch.

This is one of the reasons the project separates the **application layer** from the **underlying hardware and persistent data**.

### Reduced Host Exposure

Docker also allows services to communicate internally without publishing every application port directly to the host.

For this homelab, this is particularly important from a security perspective.

PhotoPrism and MariaDB do not need to be independently accessible from the LAN or Internet. Their communication can remain within the Docker environment while Caddy provides the controlled application entry point.

## Current Architecture

The current application stack consists of three primary containers:

```text
                         Internet
                            │
                            ▼
                    Tailscale Funnel
                            │
                            ▼
                    Host loopback :80
                            │
                            ▼
                       ┌─────────┐
                       │  Caddy  │
                       │ Reverse │
                       │  Proxy  │
                       └────┬────┘
                            │
                       Docker Network
                            │
                            ▼
                     ┌─────────────┐
                     │ PhotoPrism  │
                     │    :2342    │
                     └──────┬──────┘
                            │
                       Docker Network
                            │
                            ▼
                     ┌─────────────┐
                     │   MariaDB   │
                     │    :3306    │
                     └─────────────┘
```

## PhotoPrism

PhotoPrism is the primary application running in the Docker environment.

It provides the self-hosted interface for managing and browsing the homelab's family photo and video library.

The PhotoPrism container is intentionally not published directly to the host network.

Instead, requests are routed through Caddy.

This provides a single controlled entry point rather than exposing the application's internal port independently.

## MariaDB

MariaDB provides the database backend used by PhotoPrism.

The database is separated into its own container rather than being installed directly onto the Pop!_OS host.

PhotoPrism communicates with MariaDB through Docker's internal networking.

MariaDB does not need to be directly accessible from the LAN or Internet for normal PhotoPrism operation.

This reduces unnecessary network exposure.

## Caddy

Caddy serves as the reverse proxy for the PhotoPrism application.

Its responsibilities include:

* Receiving incoming web requests
* Forwarding requests to PhotoPrism
* Providing a controlled application entry point
* Working with the homelab's remote-access architecture

The Caddy container currently publishes HTTP to the host's loopback interface:

```text
127.0.0.1:80 → Caddy:80
```

This prevents Caddy's HTTP listener from being directly published on all host network interfaces.

External access is provided through the Tailscale Funnel layer.

## Docker Networking

Caddy and PhotoPrism communicate through a dedicated Docker network.

This allows containers to communicate using Docker's internal networking instead of requiring each service to expose its application port to the host.

The architecture can therefore be viewed as two separate networking layers:

```text
External / Host Network
        │
        ▼
   Tailscale Funnel
        │
        ▼
    Caddy :80
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

This separation is intentional: the host handles external connectivity and security controls, while the containers handle application-to-application communication.

## Storage

PhotoPrism's original media library is stored outside the container.

The current library is located on the server's storage array:

```text
/media/abrvn-storage/Pictures
```

This directory is mounted into the PhotoPrism container as its originals directory:

```text
/media/abrvn-storage/Pictures:/photoprism/originals
```

Keeping the photo library outside the container provides an important separation between **application software** and **persistent user data**.

The PhotoPrism container can therefore be replaced without inherently replacing the underlying photo library.

## Persistent Data

Container instances themselves should generally be considered replaceable.

Persistent data is treated differently.

The homelab separates:

* Container images
* Container configuration
* Application databases
* Photo and video originals
* Generated application data
* Backup data

This separation allows the application layer to be rebuilt while preserving important data through the homelab's backup and recovery strategy.

## Container Lifecycle

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

This model has also become useful as a learning tool. Rather than treating Docker as a black box, the homelab is used to understand how images, containers, networks, volumes, dependencies, and host resources interact.

## Security Considerations

Docker is treated as an **application deployment and isolation mechanism**, not as a complete security boundary.

The homelab uses additional controls, including:

* UFW host firewall
* Tailscale remote connectivity
* Restricted administrative services
* Minimal host port publishing
* Docker network isolation
* Reverse proxy architecture
* Application authentication
* Regular software updates
* Backup and recovery procedures

The goal is to reduce unnecessary exposure while maintaining reliable service access.

## Current Container State

| Container  | Purpose                      | Host Port Exposure |
| ---------- | ---------------------------- | ------------------ |
| PhotoPrism | Photo management application | None               |
| MariaDB    | PhotoPrism database          | None               |
| Caddy      | Reverse proxy                | `127.0.0.1:80`     |

PhotoPrism and MariaDB expose their application ports within the Docker environment rather than directly publishing them to the host.

Caddy provides the controlled host-facing entry point.

## Configuration Management

Production configuration files may contain sensitive information such as:

* Database credentials
* Application secrets
* Authentication information
* Private network details
* TLS-related configuration

These values should **never** be committed to the public GitHub repository.

The repository will instead contain sanitized example configurations where useful.

For example:

```text
infrastructure/
└── photoprism/
    ├── compose.example.yml
    └── README.md
```

Example files should contain placeholders rather than real passwords, tokens, private keys, or other credentials.

## Backup and Recovery

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

## Lessons Learned

Building the PhotoPrism environment has also made Docker a practical learning exercise.

One of the important lessons has been that containerization does not eliminate the need to understand the underlying operating system.

The containers still depend on:

* Host storage
* File permissions
* Networking
* DNS
* Firewall rules
* CPU and memory resources
* Kernel functionality
* Backup and recovery procedures

When something goes wrong, understanding the relationship between the host and containers is often more useful than simply restarting the container.

This project therefore uses Docker both as an infrastructure technology and as a way to develop practical Linux systems-administration skills.

## Future Improvements

Planned Docker improvements include:

* Sanitized Docker Compose examples for GitHub
* Container health monitoring
* Automated service-status reporting
* Resource monitoring
* Documented container update procedures
* Improved backup verification
* Documented disaster-recovery procedures
* Additional Docker security hardening
* Better separation of production and development configurations
* Expansion to additional self-hosted services

## Design Philosophy

The Docker architecture follows a simple principle:

> **Treat containers as replaceable application infrastructure and persistent data as something that must be deliberately protected.**

The broader goal of the ABRVN Homelab is not simply to run services. It is to build, document, break, troubleshoot, secure, and improve a real self-hosted environment while developing practical infrastructure and cybersecurity skills.
