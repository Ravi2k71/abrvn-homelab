# Docker

## Overview

Docker is used as the primary application deployment platform in the ABRVN Homelab.

Rather than installing every application directly onto the Pop!_OS host, application services are deployed as containers. This provides greater separation between the host operating system and individual applications while making services easier to deploy, update, troubleshoot, and reproduce.

The current Docker environment consists primarily of:

* PhotoPrism
* MariaDB
* Caddy

These services work together to provide the homelab's self-hosted photo management platform.

## Why Docker?

Containers provide several advantages for this homelab.

### Application Isolation

Each application runs within its own container environment instead of being installed directly onto the host operating system.

This reduces the amount of application-specific software installed on the host and helps prevent configuration conflicts between services.

For example, PhotoPrism and MariaDB have different software requirements, but they can run independently while communicating through Docker networking.

### Reproducibility

Docker Compose allows the application architecture to be described declaratively.

Instead of manually installing and configuring every component, the services, networks, volumes, and dependencies can be defined in a Compose configuration.

This makes it easier to rebuild the application stack after a system failure or migration.

### Easier Maintenance

Containers allow individual services to be updated or restarted independently.

For example, Caddy can be restarted without requiring PhotoPrism or MariaDB to be restarted.

This reduces the impact of maintenance operations on unrelated services.

### Portability

The containerized architecture can potentially be moved to another Linux host with relatively little modification.

This is useful for future hardware upgrades, disaster recovery, or migration to another server.

### Reduced Host Exposure

Docker also allows services to communicate internally without publishing every application port directly to the host.

This is an important part of the homelab's security design.

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

It provides:

* Photo and video organization
* Search and browsing
* Metadata processing
* Thumbnail generation
* Facial and image classification capabilities
* Web-based access to the photo library

The PhotoPrism container is not directly published to the host network.

Instead, requests are routed through Caddy.

This prevents the PhotoPrism application port from becoming an independently exposed host service.

## MariaDB

MariaDB provides the database backend used by PhotoPrism.

The database is separated into its own container rather than being installed directly on the Pop!_OS host.

PhotoPrism communicates with MariaDB through Docker's internal networking.

MariaDB does not need to be directly accessible from the LAN or Internet for normal PhotoPrism operation.

This reduces unnecessary network exposure.

## Caddy

Caddy serves as the reverse proxy for the PhotoPrism application.

Its responsibilities include:

* Receiving incoming HTTP/HTTPS requests
* Forwarding requests to PhotoPrism
* Handling the reverse-proxy layer
* Providing a controlled entry point to the application

The Caddy container publishes HTTP to the host's loopback interface:

```text
127.0.0.1:80 → Caddy:80
```

This means the service is bound to the server itself rather than listening directly on all LAN interfaces.

External access is provided through the Tailscale Funnel layer.

## Docker Networking

Caddy and PhotoPrism communicate through a dedicated Docker network.

This allows containers to communicate using Docker's internal networking rather than requiring each service to expose its application port to the host.

The architecture can therefore be thought of as two separate networking layers:

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

The container can be replaced without deleting the underlying photo library.

## Persistent Data

Not all application data should be treated as disposable container data.

The homelab therefore separates:

* Container images
* Container configuration
* Application databases
* Photo/video originals
* Generated application data
* Backup data

Persistent data is stored using host-mounted storage or Docker volumes where appropriate.

This allows the application layer to be recreated without necessarily recreating the underlying data.

## Container Lifecycle

The basic lifecycle of a service is:

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

When maintenance is required, individual services can be restarted or recreated without necessarily affecting the entire host system.

## Security Considerations

Docker is treated as an application isolation and deployment mechanism, **not as a complete security boundary**.

The homelab uses several additional controls:

* UFW host firewall
* Tailscale remote connectivity
* Restricted administrative services
* Minimal host port publishing
* Docker network isolation
* Reverse proxy architecture
* Application authentication
* Regular software updates
* Backup and recovery procedures

The goal is to minimize unnecessary exposure while maintaining reliable service access.

## Current Container State

The primary Docker services are currently:

| Container  | Purpose                      | Host Port Exposure |
| ---------- | ---------------------------- | ------------------ |
| PhotoPrism | Photo management application | None               |
| MariaDB    | PhotoPrism database          | None               |
| Caddy      | Reverse proxy                | `127.0.0.1:80`     |

PhotoPrism and MariaDB expose their application ports only within the Docker environment.

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

The homelab's backup strategy protects important persistent data and configuration separately from the running containers.

A recovery process should be capable of:

1. Reinstalling the operating system if necessary
2. Installing Docker
3. Restoring the Compose configuration
4. Restoring required environment variables and secrets securely
5. Restoring persistent application data
6. Starting the container stack
7. Verifying application functionality

This approach makes the container environment **rebuildable** rather than dependent on the continued existence of a particular container instance.

## Future Improvements

Planned Docker improvements include:

* Sanitized Docker Compose examples for GitHub
* Container health monitoring
* Automated service-status reporting
* Resource monitoring
* Container update strategy
* Improved backup verification
* Documented disaster-recovery procedures
* Additional Docker security hardening
* Better separation of production and development configurations
* Expansion to additional self-hosted services

## Design Philosophy

The Docker architecture follows a simple principle:

> **Treat containers as replaceable application infrastructure and persistent data as something that must be deliberately protected.**

This allows the application stack to evolve without unnecessarily tying the underlying data to a specific container installation.
