# Operating System

The ABRVN Homelab server currently runs **Pop!_OS 24.04 LTS**.

The operating system provides the foundation for the homelab's applications, storage, networking, security controls, administration tools, and monitoring systems.

Rather than treating the operating system as simply a way to launch applications, this project uses it as part of the infrastructure itself. Understanding how Linux services, permissions, networking, storage, processes, and system configuration interact is a major part of the homelab.

---

## System Configuration

| Component                        | Configuration     |
| -------------------------------- | ----------------- |
| Operating System                 | Pop!_OS 24.04 LTS |
| Desktop Environment              | GNOME             |
| Display Server                   | X11               |
| Architecture                     | 64-bit            |
| Primary Administration Interface | Webmin            |
| Container Platform               | Docker            |
| Firewall                         | UFW               |

The graphical desktop is retained because the machine is also used as an administration and troubleshooting workstation. Most persistent infrastructure services, however, are designed to operate independently of the graphical session.

---

# Why Pop!_OS?

The server originally went through a different Linux configuration before being migrated to Pop!_OS.

That process became an important part of the project.

The goal was not simply to find a distribution that could boot the hardware. The operating system needed to provide a stable foundation for:

* Docker
* PhotoPrism
* MariaDB
* Samba
* RAID storage
* Caddy
* Tailscale
* Webmin
* UFW
* Hardware monitoring
* Backup operations
* Future infrastructure development

Pop!_OS provided a familiar Ubuntu-based environment while remaining suitable for the workstation hardware being repurposed as the server.

The migration also reinforced an important infrastructure principle:

> **The best operating system for a homelab is not necessarily the one with the most features. It is the one that provides a stable foundation for the workloads the system actually needs to run.**

---

# Role in the Homelab

Linux is responsible for coordinating nearly every layer of the server.

```text id="v6p3e7"
                    Linux Host
                        │
        ┌───────────────┼────────────────┐
        │               │                │
        ▼               ▼                ▼
     Storage         Networking       Applications
        │               │                │
      RAID            UFW             Docker
      Filesystems     Tailscale       PhotoPrism
      SMART           Networking      MariaDB
      Backups         Firewall        Caddy
        │                                │
        └──────────────┬─────────────────┘
                       ▼
                  Administration
                       │
                    Webmin / SSH
```

The operating system therefore acts as the control layer connecting the physical hardware with the applications running on top of it.

---

# Application and Service Stack

The host provides the environment for several different services.

### Application Layer

* PhotoPrism
* MariaDB

### Container Layer

* Docker
* Docker Compose

### Networking Layer

* Tailscale
* Caddy
* Host networking configuration

### File and Storage Layer

* Linux filesystems
* RAID
* Samba
* SMART monitoring
* Backup processes

### Administration Layer

* Webmin
* SSH
* systemd
* UFW

### Monitoring Layer

* System resource monitoring
* Service health checks
* Hardware telemetry
* Custom Webmin dashboard development

This layered architecture allows individual services to be changed without rebuilding the entire server.

---

# Linux Administration Approach

The server is intentionally managed using both graphical and command-line tools.

**Webmin** provides the primary graphical administration interface for routine system management.

The command line remains important for:

* Troubleshooting
* Service inspection
* Log analysis
* Network diagnostics
* Storage administration
* Docker management
* Configuration validation
* Hardware monitoring
* Automation

This hybrid approach is intentional.

A graphical management interface makes routine administration easier, but relying exclusively on a GUI can hide important system behavior.

The project therefore uses Webmin as an administration layer while maintaining an understanding of the underlying Linux services and configuration.

---

# systemd and Persistent Services

Linux `systemd` is responsible for managing many of the server's persistent services and background processes.

This is important for a server because applications need to continue operating without requiring manual intervention after every reboot.

Examples include:

* Docker
* Samba
* Tailscale
* Webmin
* Other infrastructure services
* Custom monitoring and automation components

Service dependencies and startup behavior are therefore treated as part of the infrastructure design rather than an afterthought.

---

# Storage Integration

The operating system also provides the storage layer for the homelab.

Linux manages:

* RAID devices
* Filesystems
* Mount points
* File permissions
* Storage availability
* SMART information
* Network file sharing
* Backup access

This is particularly important because PhotoPrism depends on persistent media storage.

The application layer and storage layer are therefore intentionally treated as separate concerns.

A sanitized example architecture is:

```text id="w8q0by"
PhotoPrism
    │
    ▼
Container Mount
    │
    ▼
Linux Filesystem
    │
    ▼
RAID Storage
    │
    ▼
Physical HDDs
```

This separation makes it easier to understand where a failure occurs.

For example, a PhotoPrism problem is different from a Docker problem, filesystem problem, RAID problem, or physical-drive problem.

---

# Networking Integration

The operating system provides the network stack used by the server.

The infrastructure uses multiple layers rather than exposing every application directly to the network.

```text id="8slf2x"
Remote Client
      │
      ▼
Tailscale
      │
      ▼
Linux Network Stack
      │
      ├── UFW
      │
      ├── Docker Networking
      │
      └── Local Services
```

This layered approach allows network access to be controlled independently of individual applications.

The server does not need every application to independently manage Internet exposure.

---

# Firewall Management

**UFW** provides host-level firewall controls.

The firewall is configured around the principle of allowing only the network access required by a service.

For example, administrative services can be restricted to trusted networks while application services can use their intended access path.

This creates an additional security boundary between:

* External clients
* Remote-access networking
* The Linux host
* Containers
* Administrative interfaces

Firewall configuration is therefore considered part of the operating system configuration rather than merely an application setting.

---

# Docker and Host Separation

Docker allows application services to be isolated from the host environment.

The server uses containers for application workloads such as PhotoPrism, its database, and the reverse-proxy stack.

A simplified model is:

```text id="e6m3u2"
Linux Host
    │
    └── Docker
         │
         ├── PhotoPrism
         ├── MariaDB
         └── Caddy
```

This separation provides several advantages:

* Applications can have independent configurations
* Application dependencies are isolated
* Services can be restarted independently
* Host ports can be minimized
* Application updates can be managed separately
* Infrastructure can be reproduced more easily

The host remains responsible for the underlying hardware, storage, networking, firewall, and system services.

---

# Lessons From Earlier Linux Configuration

One of the most valuable parts of the project has been dealing with the problems that occurred while building the server.

Earlier configurations exposed several practical Linux administration challenges involving:

* Permissions
* Filesystem mounts
* Boot configuration
* Remote administration
* Desktop/server interaction
* Hardware drivers
* Container networking
* Service dependencies
* Configuration management

These problems demonstrated that Linux infrastructure is highly interconnected.

A change that appears isolated can affect several other layers.

For example:

```text
Storage configuration
        │
        ▼
Filesystem mount
        │
        ▼
Permissions
        │
        ▼
Container access
        │
        ▼
Application availability
```

Understanding these relationships became more important than simply knowing which command fixes an individual problem.

---

# Configuration and Reliability Philosophy

The server is treated as infrastructure rather than a disposable desktop.

Before changing an important component, the potential effect on the rest of the system is considered.

Areas that require particular caution include:

* Boot configuration
* Filesystem mounts
* RAID configuration
* Network configuration
* Firewall rules
* Docker networking
* Service dependencies
* Authentication
* Remote administration

The goal is to avoid solving one problem by creating another.

This also led to a preference for:

* Reproducible configuration
* Configuration validation before restarting services
* Backups before major changes
* Testing changes incrementally
* Keeping documentation alongside infrastructure
* Version-controlling reproducible configuration
* Using monitoring to verify the result

---

# Why the Desktop Environment Is Still Present

Although the system functions as a server, the graphical environment is intentionally retained.

There are practical reasons for this.

The machine is also used for:

* Hardware troubleshooting
* Driver investigation
* GPU monitoring
* Webmin development
* Local administration
* Testing graphical tools
* Diagnosing problems that may be difficult to investigate remotely

This makes the system closer to a **workstation-server hybrid** than a traditional headless server.

The distinction is useful because it allows the same physical machine to function as both infrastructure and a development/testing environment.

---

# Hardware and Linux Integration

The operating system also provides the interface between the older workstation hardware and the applications running on it.

This includes:

* CPU scheduling
* Memory management
* Storage drivers
* RAID management
* Network interfaces
* GPU drivers
* Hardware telemetry
* Filesystem operations

The older Intel integrated GPU is particularly relevant because Linux exposes GPU-engine telemetry that can be incorporated into the custom monitoring system.

This makes the operating system part of the hardware-observability layer.

Rather than treating the hardware as a black box, the project uses Linux interfaces to inspect how the system is actually operating.

---

# Monitoring and Observability

One of the longer-term goals is to make the operating system's internal state visible through the custom Webmin dashboard.

The dashboard is intended to bring together information from multiple Linux subsystems, including:

* CPU utilization
* Memory usage
* GPU activity
* RAID health
* Storage capacity
* SMART health
* Docker status
* Service status
* Network connectivity
* Tailscale status
* Firewall state
* Backup status

This creates an observability layer above the individual Linux commands and interfaces.

The goal is not simply to display numbers.

The dashboard should help answer:

> **Is the server healthy, and if it is not, where should I investigate first?**

---

# Software Management

System software and security updates are managed through the operating system's package-management system.

Docker is maintained separately using its appropriate package source.

Third-party software repositories are kept to a minimum and should be documented when required.

Software changes are evaluated based on their effect on:

* Compatibility
* Security
* Service availability
* Container behavior
* Hardware support
* Recovery procedures

This is especially important on an older hardware platform where newer software can sometimes introduce compatibility considerations.

---

# Reliability and Recovery

The operating system is one component of the server's overall recovery strategy.

A recoverable system requires more than simply backing up application data.

Important recovery components include:

* Application data
* Configuration
* Docker deployment definitions
* Storage configuration
* Network configuration
* Firewall configuration
* Authentication configuration
* Service definitions
* Documentation

The project therefore treats configuration and documentation as infrastructure assets.

A server should be rebuildable from documented information rather than depending entirely on undocumented changes made manually over time.

---

# Security Considerations

The operating system provides several of the server's security boundaries.

Current security practices include:

* Host-based UFW firewall controls
* Restricted administrative services
* Tailscale for secure remote connectivity
* Limited host port exposure
* Container isolation
* Regular software updates
* Separation between administrative and application access
* Backup and recovery planning

Security configuration is treated as part of the infrastructure design rather than something added after the services are deployed.

---

# Future Operating-System Work

Planned improvements include:

* More comprehensive system monitoring
* Hardware health monitoring
* Expanded service health checks
* Automated reporting
* Configuration backup
* Improved disaster-recovery documentation
* Additional firewall hardening
* Improved authentication controls
* Tailscale access-policy improvements
* Evaluation of additional self-hosted workloads
* Possible virtualization experiments
* Continued refinement of the custom Webmin dashboard

The operating system will remain the foundation for these experiments.

---

# Operating-System Philosophy

The Linux installation is not simply the software that makes the server boot.

It is the layer connecting:

**hardware → storage → networking → security → containers → applications → monitoring**

The experience of building this server has reinforced the importance of understanding those relationships.

The goal is therefore not merely to learn how to administer Pop!_OS.

It is to understand how a Linux system behaves when it becomes a real piece of infrastructure with persistent data, network services, security boundaries, application dependencies, and recovery requirements.

That understanding is one of the primary purposes of the homelab.
