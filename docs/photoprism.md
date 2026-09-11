# PhotoPrism

## Overview

PhotoPrism is the primary self-hosted application deployed in the ABRVN Homelab.

The project uses PhotoPrism to provide a private, browser-based interface for organizing, indexing, searching, and viewing a personal photo and video collection without relying entirely on a third-party cloud photo platform.

PhotoPrism runs as a Docker container and uses MariaDB as its database backend.

The application is integrated with the rest of the homelab through Docker networking, Caddy, Tailscale, host firewall controls, persistent storage, and the backup system.

---

## Why PhotoPrism?

The original goal was to build a self-hosted photo platform that could provide remote access while keeping the underlying infrastructure under local control.

PhotoPrism was selected because it provides several capabilities useful for this project:

* Self-hosted photo and video management
* Browser-based access
* Metadata indexing
* Search and organization
* Docker-based deployment
* Database-backed application management
* Support for large media collections
* Separation between application software and stored media

The project also provided a practical opportunity to learn how a real self-hosted application depends on multiple layers of infrastructure.

---

## Application Architecture

PhotoPrism is not deployed as a standalone container.

It is part of a larger application stack:

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

The components have distinct responsibilities.

### PhotoPrism

Provides the primary photo-management application and user interface.

### MariaDB

Provides the database backend used by PhotoPrism.

### Caddy

Acts as the reverse proxy and controlled entry point for the application.

### Tailscale

Provides the remote connectivity infrastructure used by the homelab. Public PhotoPrism access is provided through the Tailscale Funnel architecture.

### Linux Host

Provides the underlying operating system, storage, firewall, Docker runtime, and hardware resources.

---

## Docker Deployment

PhotoPrism runs inside Docker rather than being installed directly onto the Linux host.

The deployment uses Docker Compose to define the application and its dependencies.

The PhotoPrism container communicates with MariaDB through Docker's internal networking.

Neither PhotoPrism nor MariaDB needs to expose its application port directly to the host.

This results in a layered architecture:

```text
Linux Host
    │
    ▼
  Docker
    │
    ├───────────────┐
    │               │
    ▼               ▼
PhotoPrism       MariaDB
    │
    ▼
Docker Network
```

Caddy connects to PhotoPrism through the Docker networking layer rather than requiring PhotoPrism's application port to be publicly exposed.

This design reduces unnecessary host-level exposure.

---

## Persistent Storage

The original photo and video collection is stored outside the PhotoPrism container.

A sanitized representation of the storage architecture is:

```text
PhotoPrism Container
        │
        ▼
  Mounted Originals
        │
        ▼
 Host Storage
        │
        ▼
 Storage Array
```

Keeping original media outside the container is important because the container itself should be considered replaceable.

If PhotoPrism needs to be recreated or upgraded, the underlying media should remain independent from the container lifecycle.

This creates a deliberate separation between:

* Application software
* Application configuration
* Database data
* Original media
* Generated application data
* Backups

---

## Database

PhotoPrism uses MariaDB as its database backend.

The database operates as a separate Docker container and communicates with PhotoPrism through an internal Docker network.

The database is not intended to be directly accessible from the public Internet.

This design provides both organizational and security benefits:

```text
PhotoPrism
     │
     │ Internal Docker Network
     ▼
  MariaDB
```

Keeping the database internal prevents the database service from becoming an unnecessary externally exposed component.

---

## Photo and Video Indexing

One of the most significant parts of the PhotoPrism deployment was the initial indexing process.

The server was required to process a large existing media collection rather than starting with an empty library.

This provided a useful real-world performance test for the hardware.

The initial library contained approximately **546 GB** of mixed photos and videos.

The initial indexing process completed in approximately **8 hours and 3 minutes** with no reported indexing errors.

This was useful for establishing a baseline for the performance of the current hardware.

The system is based on an older quad-core Intel processor and 16 GB of RAM, so the indexing workload also demonstrated the practical limitations of older hardware when processing a large media collection.

---

## Indexing Considerations

PhotoPrism indexing is more than simply scanning filenames.

A large media library can involve processing:

* File metadata
* Image metadata
* Video metadata
* Thumbnails
* Search information
* Application database records
* Sidecar information
* Generated application data

Because of this, indexing can place sustained load on the server.

The project therefore treats indexing as an infrastructure workload rather than assuming that the application will have negligible resource requirements.

This is particularly relevant when monitoring:

* CPU utilization
* Memory usage
* Storage activity
* Disk capacity
* Database performance

These metrics will eventually become part of the custom Webmin dashboard.

---

## Remote Access

Remote access was one of the more challenging parts of deploying PhotoPrism.

The home Internet connection uses carrier-grade NAT, which prevents the server from being reliably exposed using a conventional public IPv4 port-forwarding configuration.

Instead of working around the limitation by directly exposing additional services, the project moved toward a Tailscale-based architecture.

The current conceptual access path is:

```text
Internet
    │
    ▼
Tailscale Funnel
    │
    ▼
Host Loopback
    │
    ▼
Caddy
    │
    ▼
PhotoPrism
```

This allows PhotoPrism to be remotely accessible without directly exposing the server's application ports through the home router.

---

## Caddy Integration

Caddy provides the reverse-proxy layer between the remote access infrastructure and PhotoPrism.

The Caddy container exposes only the required host-facing entry point.

PhotoPrism remains inside the Docker network.

The resulting architecture is:

```text
Remote Access
      │
      ▼
    Caddy
      │
      ▼
Docker Network
      │
      ▼
PhotoPrism
      │
      ▼
  MariaDB
```

This architecture also allows the remote-access layer to remain independent from the internal application topology.

---

## Authentication

PhotoPrism uses application-level authentication to control access to the library.

Administrative and normal user access are treated as separate concerns from the underlying server administration.

The project also considers multi-factor authentication an important part of protecting remotely accessible accounts.

Credentials are intentionally excluded from this repository.

No passwords, tokens, authentication secrets, or private account information should be committed to GitHub.

---

## Security Model

PhotoPrism is publicly reachable through the remote-access layer, so the application must be treated as an Internet-facing service.

Security therefore does not depend on Docker alone.

The deployment uses multiple layers of protection:

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
Docker Network
    │
    ├── PhotoPrism
    │
    └── MariaDB
```

Additional host-level controls include:

* UFW firewall rules
* Restricted administrative services
* Minimal host port exposure
* Docker network isolation
* Application authentication
* Secure remote administration
* Regular updates
* Backup and recovery procedures

The goal is to follow a **defense-in-depth** approach rather than relying on a single security mechanism.

---

## What Is Not Publicly Documented

Because this repository is public, sensitive deployment information is intentionally excluded.

The repository should not contain:

* Real filesystem paths
* Private IP addresses
* Tailscale device identifiers
* Public/private infrastructure hostnames
* Domain names associated with private infrastructure
* Usernames
* Passwords
* Database credentials
* Authentication tokens
* TLS private keys
* Personal photos or videos
* Private backup locations

Where configuration examples are useful, they should use sanitized placeholders.

For example:

```text
/srv/photos
```

may be used as a generic example instead of documenting the actual storage location.

---

## Backup Considerations

PhotoPrism is only one component of the data-protection strategy.

The photo collection, application database, configuration, and other important data must be considered separately when designing recovery procedures.

The project follows a broader backup strategy rather than treating RAID or Docker as a backup mechanism.

The important distinction is:

> **Availability and redundancy are not the same thing as backup.**

PhotoPrism's ability to display a library does not guarantee that the underlying data can be recovered after hardware failure or accidental deletion.

---

## Recovery Model

A complete PhotoPrism recovery process should be capable of rebuilding the application environment.

A simplified recovery sequence is:

```text
Restore Host
    │
    ▼
Install Docker
    │
    ▼
Restore Configuration
    │
    ▼
Restore Database
    │
    ▼
Restore Original Media
    │
    ▼
Start PhotoPrism
    │
    ▼
Verify Application
```

The goal is not to preserve a particular container instance.

The goal is to preserve the information required to recreate the service.

---

## Lessons Learned

The PhotoPrism deployment became significantly more than an application installation exercise.

It exposed several relationships between different areas of systems administration.

### Application Problems Can Be Infrastructure Problems

A PhotoPrism issue may actually originate from:

* Docker networking
* Filesystem permissions
* Storage availability
* Database connectivity
* DNS
* Reverse-proxy configuration
* Firewall rules
* Remote-access configuration

Troubleshooting therefore requires understanding the entire application path.

### Remote Access Requires Architectural Thinking

The original approach of trying to use traditional port forwarding was complicated by the ISP's CGNAT environment.

Rather than treating the problem as simply "open another port," the project eventually moved toward a different architecture using Tailscale.

This demonstrated an important infrastructure principle:

> **When a network constraint cannot be removed, redesign the access path around the constraint.**

### Storage Must Be Separated From Applications

Keeping the original media outside the application container makes the application easier to rebuild and migrate.

This also makes the backup strategy easier to reason about because the application and its important persistent data can be treated as separate components.

### Security Must Be Layered

Docker isolation by itself is not sufficient.

The PhotoPrism deployment demonstrated why security needs to consider:

```text
Application
    +
Container
    +
Host
    +
Network
    +
Authentication
    +
Backups
```

A weakness at any one layer can affect the overall system.

---

## Current Status

The PhotoPrism deployment is operational and has successfully processed the existing media collection.

The current architecture includes:

* PhotoPrism CE
* MariaDB
* Docker Compose
* Caddy reverse proxy
* Tailscale remote access
* Persistent external media storage
* Host firewall controls
* Application authentication
* Backup and recovery planning

The application has also been tested through remote access, demonstrating that the complete access path works beyond the local network.

---

## Future Improvements

Planned improvements include:

* Additional PhotoPrism monitoring
* Application health checks
* Integration with the custom Webmin dashboard
* Automated service-status reporting
* Better backup verification
* Documented PhotoPrism recovery procedures
* Periodic security review
* Improved authentication controls
* Further remote-access hardening
* Performance monitoring during indexing and maintenance
* Documentation of future upgrades and migration procedures

---

## Project Perspective

PhotoPrism is the application that brought many of the homelab's infrastructure components together.

Running the application required solving problems involving:

* Linux
* Docker
* Databases
* Storage
* Networking
* Reverse proxies
* Remote access
* Firewalls
* Authentication
* Backups
* Troubleshooting

Because of this, PhotoPrism serves as more than a self-hosted photo application within the project.

It acts as a practical example of how multiple infrastructure layers must work together to provide a reliable service.

The long-term goal is to continue improving the system while documenting not only **what was configured**, but also **why each architectural decision was made and what was learned from implementing it**.
