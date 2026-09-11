# Hardware

The ABRVN Homelab server is built on a Dell Precision 3620 workstation. The system is being repurposed as a self-hosted server for storage, photo management, virtualization/container workloads, monitoring, and security experimentation.

## Server

| Component | Specification |
|---|---|
| System | Dell Precision 3620 |
| CPU | Intel Core i5-6500 |
| RAM | 16 GB |
| GPU | Intel integrated graphics |
| Storage | 2 × 1 TB HDD |
| Storage Configuration | RAID 0 |
| Operating System | Pop!_OS 24.04 LTS |

## Why the Precision 3620?

The Precision 3620 provides a relatively inexpensive workstation platform for experimenting with self-hosted infrastructure. Its desktop-class hardware provides enough CPU, memory, storage connectivity, and expansion capability for the current homelab workload while keeping power consumption and hardware costs relatively modest.

The system is currently being used for:

- PhotoPrism photo management
- Docker container workloads
- RAID storage
- Samba file sharing
- Caddy reverse proxy
- Tailscale networking
- Webmin system administration
- System monitoring and automation
- Security and infrastructure experimentation

## Storage

The server currently uses two 1 TB hard drives configured as a RAID 0 array.

RAID 0 provides increased storage capacity and potentially improved sequential performance, but provides **no redundancy**. A failure of either drive can result in loss of the entire array.

RAID is therefore not treated as a backup mechanism. Important data is protected through the homelab's separate backup strategy.

## Future Hardware Plans

Potential future improvements include:

- Replacing the current RAID 0 array with RAID 1
- Increasing storage capacity
- Adding dedicated backup storage
- Evaluating drive health and SMART monitoring
- Improving server hardware as the homelab grows
