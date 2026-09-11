# Hardware

The ABRVN Homelab server is built around a repurposed **Dell Precision 3620 workstation**. Rather than using the system for a single application, the goal is to use it as a centralized home infrastructure server capable of hosting persistent services, managing large amounts of media, providing network storage, and serving as a platform for learning Linux, Docker, networking, security, monitoring, and system administration.

The hardware is older by modern standards, so part of the project has been learning how to determine what the platform can realistically handle and how to compensate for its limitations through software architecture and infrastructure design.

## Server Specifications

| Component             | Specification                            |
| --------------------- | ---------------------------------------- |
| System                | Dell Precision 3620                      |
| CPU                   | Intel Core i5-6500                       |
| CPU Architecture      | 6th-generation Intel Skylake             |
| CPU Cores / Threads   | 4 / 4                                    |
| RAM                   | 16 GB                                    |
| GPU                   | Intel Skylake integrated graphics / Gen9 |
| Primary Storage       | 2 × 1 TB HDD                             |
| Storage Configuration | RAID 0                                   |
| Operating System      | Pop!_OS 24.04 LTS                        |

---

# Why This Hardware?

The server was not intended to be a high-performance compute system.

The goal was to repurpose available workstation hardware into a machine capable of handling the following types of workloads:

* Self-hosted photo management
* Large photo and video storage
* Network file sharing
* Docker containers
* Database services
* Reverse proxying
* Secure remote access
* System administration
* Hardware and service monitoring
* Automated backups
* Security experimentation
* Future infrastructure projects

The Precision 3620 provides enough CPU, memory, storage connectivity, and expansion capability to accomplish those goals without requiring a dedicated enterprise server.

This makes it particularly useful as a homelab platform because the system has enough capability to run meaningful infrastructure while still having visible hardware limitations that require engineering decisions.

---

# Intended Workload

The server is primarily designed around **persistent infrastructure workloads**, rather than high-performance computing.

The workload can broadly be divided into several layers:

```text
                    Home Infrastructure Server
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
          ▼                   ▼                   ▼
     Applications          Storage            Administration
          │                   │                   │
     PhotoPrism           RAID Storage          Webmin
     MariaDB              Samba                SSH
     Caddy                Backups              Monitoring
          │                   │                   │
          └───────────────────┼───────────────────┘
                              ▼
                     Networking & Security
                              │
                    Tailscale / UFW / Docker
```

The architecture intentionally favors services that can run continuously with relatively predictable resource requirements.

---

# PhotoPrism as the Primary Workload

PhotoPrism is the most significant application workload on the server.

The purpose of the system is to maintain a self-hosted photo and video library rather than relying entirely on a cloud-based service.

This creates several different hardware requirements:

### CPU

PhotoPrism can perform CPU-intensive work during:

* Library indexing
* Image processing
* Thumbnail generation
* Metadata processing
* Video handling
* Search/index operations

The i5-6500 is capable of handling these workloads, but it is not a modern high-core-count processor.

A large library can therefore result in long indexing periods.

This became an important practical lesson in the project:

> **A workload being capable of running on hardware does not mean it will run quickly on that hardware.**

The server therefore prioritizes reliability and predictable operation over minimizing processing time.

### Memory

PhotoPrism, MariaDB, Docker, the operating system, filesystem caching, and other services all share the available 16 GB of RAM.

This makes memory management important when several workloads operate simultaneously.

Instead of filling the system with unnecessary virtual machines and heavy services, the architecture uses lightweight containers and persistent services that provide direct value to the homelab.

### Storage

PhotoPrism is particularly dependent on storage performance because the system needs to read and process a large number of files.

The mechanical HDD array provides inexpensive capacity, which is valuable for a photo and video library.

The tradeoff is significantly higher latency compared with SSD storage.

This means operations involving many small files can become storage-bound even when CPU utilization is relatively low.

---

# CPU Limitations and How They Are Managed

The i5-6500 provides four physical cores and four threads.

That is enough for the current infrastructure, but it creates a finite amount of parallel processing capacity.

The limitation becomes most apparent when several CPU-heavy operations occur simultaneously.

For example:

```text
PhotoPrism indexing
        +
Database activity
        +
File transfers
        +
Docker services
        +
Operating system
        =
Higher CPU contention
```

The server therefore does not attempt to run every possible service simply because Docker makes it easy to deploy them.

### Temporary Mitigation

The project bridges this limitation through:

* Keeping the application stack relatively lightweight
* Avoiding unnecessary virtual machines
* Separating services into containers
* Monitoring CPU utilization
* Identifying resource-intensive workloads before adding additional services
* Allowing long-running processing jobs to complete rather than requiring immediate results
* Designing future upgrades around measured bottlenecks

This makes the server suitable for persistent infrastructure even though it is not designed for large-scale computation.

---

# Integrated GPU Architecture

The Precision 3620 uses Intel's **Skylake-generation Gen9 integrated graphics architecture**.

This is fundamentally different from a modern discrete GPU.

The iGPU does not have a large pool of dedicated VRAM. Instead, it operates within the system's shared memory architecture.

This is important for a server with only 16 GB of RAM because the GPU and CPU ultimately compete for the same system resources.

The GPU therefore cannot simply be treated as an independent accelerator.

---

# Older iGPU Limitations

The Gen9 iGPU has several limitations compared with modern graphics hardware.

## Limited Compute Capability

The integrated GPU was not designed to compete with modern discrete GPUs for general-purpose parallel computation.

It is therefore poorly suited to workloads such as:

* Modern AI inference
* Machine-learning workloads
* Large-scale GPU image processing
* High-performance computer vision
* Heavy GPU compute
* Modern GPU-accelerated applications

The server's architecture therefore does not depend on the iGPU for its core functionality.

## Shared System Memory

Because the GPU uses system memory, heavy GPU workloads can compete with the operating system and applications for memory bandwidth and capacity.

For this particular server, this matters because PhotoPrism, MariaDB, Docker, filesystem caching, and the operating system all benefit from available RAM.

### Temporary Mitigation

The iGPU is treated as an **optional resource**, rather than something every application is expected to use.

The server's core workloads remain functional without requiring a modern GPU.

This allows the existing hardware to continue providing useful infrastructure while avoiding an unnecessary GPU upgrade.

---

# Media Acceleration Limitations

The Skylake iGPU includes dedicated media functionality, but its capabilities are tied to the generation of the hardware.

Newer GPUs and integrated graphics platforms provide broader support for newer codecs, formats, resolutions, and hardware acceleration paths.

This means that simply having Intel graphics does not guarantee acceleration for every modern media workload.

The practical approach on this server is therefore:

```text
Hardware acceleration available?
             │
        ┌────┴────┐
       Yes         No
        │           │
        ▼           ▼
Use when useful   CPU fallback
```

Hardware acceleration is treated as an optimization rather than a hard dependency.

That is important for long-term compatibility: an application should remain functional even when a particular acceleration API or codec is unavailable.

---

# GPU Monitoring Work

The limitations of the older iGPU also created an opportunity to build a better understanding of the hardware.

Rather than assuming the GPU was either useful or useless, the project introduced custom GPU monitoring into the Webmin dashboard.

The monitoring work reads Linux GPU telemetry and exposes GPU-engine activity through the server-management interface.

This allows the system to answer practical questions such as:

* Is the GPU actually being used?
* Which type of GPU workload is active?
* Is a workload CPU-bound or GPU-bound?
* Is GPU acceleration providing meaningful benefit?
* Is the GPU contributing to system resource pressure?
* Would a future GPU upgrade actually solve a demonstrated bottleneck?

This is an important part of the project's hardware philosophy.

**Monitoring is used to make upgrade decisions based on evidence rather than assumptions.**

---

# Storage Architecture

The server currently uses two 1 TB mechanical hard drives.

They are configured as RAID 0 to provide approximately twice the capacity of a single drive and allow the available disks to be used as a single storage pool.

The tradeoff is that RAID 0 provides no redundancy.

A failure of either drive can make the entire array unavailable.

This configuration is therefore appropriate only because the project treats the RAID array separately from the backup system.

The storage architecture is intended to provide the capacity required by the PhotoPrism library while a separate 3-2-1 backup strategy provides data protection.

---

# HDD Performance Limitations

Mechanical storage is one of the largest performance limitations of the current system.

Compared with SSDs, HDDs have:

* Higher access latency
* Lower random I/O performance
* Slower small-file operations
* Mechanical seek overhead
* Greater sensitivity to simultaneous random workloads

This is particularly relevant to PhotoPrism because a large media library can involve a substantial number of individual files.

The database also performs operations where latency can matter.

### Temporary Mitigation

The architecture separates the different storage responsibilities rather than treating all storage as one undifferentiated resource.

The large media library is stored on high-capacity HDD storage, while application services remain containerized.

This makes the inexpensive HDD capacity useful for the data that requires it without unnecessarily placing every application component directly on the large media array.

A future SSD upgrade can therefore target the parts of the workload where lower latency would provide the greatest benefit.

---

# RAM Limitations

The server has 16 GB of RAM.

For the current workload, this is enough to run the operating system, Docker, PhotoPrism, MariaDB, networking, file sharing, Webmin, and monitoring.

However, it limits how many additional heavy workloads can be introduced.

The system is therefore intentionally not designed around running numerous virtual machines.

### Temporary Mitigation

Docker provides a more lightweight way to isolate applications than deploying a separate virtual machine for every service.

This allows the available RAM to be used primarily by the actual services rather than multiple complete guest operating systems.

Monitoring also makes it possible to identify memory pressure before adding additional workloads.

---

# Networking and Hardware Interaction

The server's hardware also has to support continuous network activity.

The intended workload includes:

* Large media transfers
* Network file sharing
* Remote PhotoPrism access
* Container networking
* Backup transfers
* Remote administration

The goal is not maximum network throughput at all times.

Instead, the network architecture is designed so that services can share the available hardware resources without unnecessarily exposing individual applications.

Docker provides internal service networking, while Tailscale provides secure remote connectivity and UFW controls host-level access.

This reduces the need for every application to independently handle network exposure.

---

# Bridging the Hardware Gap

The server has remained useful because the project has compensated for its hardware limitations at multiple layers.

| Limitation                               | Engineering Response                                                   |
| ---------------------------------------- | ---------------------------------------------------------------------- |
| Older 4-core CPU                         | Keep workloads focused, use containers, monitor utilization            |
| Long CPU-intensive PhotoPrism processing | Accept longer processing times instead of requiring excessive hardware |
| 16 GB RAM                                | Prefer containers and lightweight services over numerous VMs           |
| Older integrated GPU                     | Avoid making modern GPU compute a core dependency                      |
| Shared GPU/system memory                 | Avoid unnecessary GPU-heavy workloads                                  |
| Older media engine                       | Treat acceleration as optional and retain CPU fallback                 |
| Limited GPU compute                      | Build around CPU-based infrastructure workloads                        |
| HDD storage latency                      | Use HDDs primarily for high-capacity media storage                     |
| RAID 0 has no redundancy                 | Maintain an independent 3-2-1 backup strategy                          |
| Aging hardware                           | Use monitoring to identify actual bottlenecks                          |
| Limited hardware visibility              | Develop custom Webmin hardware telemetry                               |

The result is not an attempt to make old hardware behave like new hardware.

Instead, the goal is to **design the infrastructure around what the hardware is actually capable of**.

---

# Why the Older Platform Is Still Valuable

The age of the hardware is actually useful for this project.

A modern workstation with excessive CPU, RAM, SSD performance, and GPU capability could hide poor architectural decisions because the hardware would simply absorb the inefficiency.

The Precision 3620 does not provide that luxury.

When PhotoPrism takes a long time to index, storage performance and CPU utilization become measurable engineering problems.

When multiple containers consume memory simultaneously, resource contention becomes visible.

When the iGPU cannot efficiently handle a workload, the limitation has to be understood rather than ignored.

This forces the project to consider:

* CPU architecture
* Core and thread availability
* Memory capacity
* Shared versus dedicated GPU memory
* GPU compute capability
* Media acceleration
* Driver support
* Storage latency
* I/O behavior
* Network utilization
* Power consumption
* Service isolation
* Monitoring
* Backup architecture
* Upgrade justification

That makes the hardware itself part of the learning environment.

---

# Future Hardware Upgrade Strategy

The current server is considered a **temporary but functional platform**.

The next hardware platform will be selected based on demonstrated requirements rather than simply purchasing newer hardware.

Potential upgrades include:

### Storage

* Replace RAID 0 with RAID 1 or another redundant storage design
* Increase storage capacity
* Add SSD storage for application/database workloads
* Expand dedicated backup storage

### Memory

Increase RAM if simultaneous PhotoPrism, database, Docker, monitoring, and future workloads begin creating measurable memory pressure.

### CPU

Move to a newer platform if CPU utilization becomes a consistent bottleneck, particularly during PhotoPrism processing or additional concurrent services.

### GPU

A newer iGPU or discrete GPU would only be justified if the server develops a workload that actually benefits from GPU acceleration, such as significant media transcoding, computer vision, or AI workloads.

### Platform

Eventually moving to a newer, more power-efficient platform could reduce:

* Power consumption
* Processing time
* Storage bottlenecks through newer interfaces
* Memory limitations
* GPU limitations

However, replacing the platform before those limitations materially affect the workload would provide less value than improving the architecture and monitoring first.

---

# Hardware Philosophy

The purpose of this server is not to demonstrate how much hardware can be purchased.

It is to demonstrate how much useful infrastructure can be built by **understanding the hardware that is actually available**.

The current platform supports a real self-hosted application, persistent storage, network services, remote access, containerization, administration, monitoring, and security controls.

Its limitations have also directly influenced the architecture.

The approach is:

> **Understand the hardware → identify the bottleneck → engineer around it → monitor the result → upgrade when the limitation is actually justified.**

That process is a core part of the homelab.

The Precision 3620 may be an older workstation, but it provides enough capability to function as a serious learning platform while forcing the project to develop an understanding of the relationship between **hardware architecture, operating systems, applications, storage, networking, and infrastructure design**.
