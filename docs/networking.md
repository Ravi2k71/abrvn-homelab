# Networking

## Overview

The ABRVN Homelab uses a combination of local LAN networking, Tailscale, Caddy, and host-based firewall rules to provide access to services while minimizing unnecessary exposure.

The networking design separates **public application access** from **private administrative access**. PhotoPrism is made remotely accessible through Tailscale Funnel, while administrative services such as SSH, Webmin, and Samba remain restricted to trusted networks.

## Network Environment

| Component                 | Configuration    |
| ------------------------- | ---------------- |
| LAN Network               | `192.168.4.0/22` |
| Default Gateway           | `192.168.4.1`    |
| Server LAN Address        | `192.168.7.197`  |
| Remote Access             | Tailscale        |
| Reverse Proxy             | Caddy            |
| Public Application Access | Tailscale Funnel |
| Host Firewall             | UFW              |
| ISP                       | W.O.W.           |

The home network uses a `/22` private IPv4 network, providing addresses across multiple `/24` ranges.

## Internet Connectivity

The ISP connection uses carrier-grade NAT (CGNAT), which prevents the server from being directly reachable through a traditional public IPv4 port-forwarding configuration.

The WAN address presented to the local network has historically fallen within the `100.64.0.0/10` CGNAT address space, while external services report a different public IPv4 address.

Because of this limitation, traditional direct Internet port forwarding is not used for remote PhotoPrism access.

This design also avoids exposing administrative services directly to the Internet.

## Tailscale

**Tailscale** provides the primary secure remote connectivity layer for administration and private network access.

The server is registered as:

```text
abrvn-server
```

Tailscale provides encrypted connectivity between trusted devices without requiring inbound ports to be exposed through the home router.

Remote administrative access can therefore be restricted to the Tailscale network rather than being made publicly accessible.

## PhotoPrism Remote Access

PhotoPrism is exposed remotely through **Tailscale Funnel**.

The current public access path is:

```text
Internet
    │
    ▼
Tailscale Funnel
    │
    ▼
127.0.0.1:80
    │
    ▼
Caddy
    │
    ▼
PhotoPrism
```

Caddy acts as the reverse-proxy layer between the incoming HTTPS request and the PhotoPrism container.

The Caddy container only publishes port 80 to the host's loopback interface:

```text
127.0.0.1:80 → Caddy:80
```

This prevents Caddy's HTTP listener from being directly exposed on the server's LAN interfaces.

## Docker Network Isolation

Caddy and PhotoPrism communicate through a dedicated Docker network.

PhotoPrism and MariaDB do not require their application ports to be directly published on the host.

This reduces unnecessary host-level exposure and keeps internal application communication inside the Docker environment.

The architecture is approximately:

```text
Internet
    │
    ▼
Tailscale Funnel
    │
    ▼
127.0.0.1:80
    │
    ▼
Caddy
    │
    ▼
Docker Proxy Network
    │
    ├── PhotoPrism
    │
    └── MariaDB
```

## Firewall

UFW provides host-level access control.

Current firewall rules restrict administrative and file-sharing services to trusted networks.

Examples include:

* SSH restricted to trusted LAN and Tailscale access
* Webmin restricted to the local LAN
* Samba restricted to the local LAN
* Tailscale networking allowed through its required interface/port
* Application services not unnecessarily published to the host

The firewall is treated as an additional security boundary rather than the only security control.

## Administrative Access

Administrative services are intentionally separated from the public PhotoPrism access path.

### SSH

SSH is available for system administration and troubleshooting but is restricted through UFW.

### Webmin

Webmin provides graphical Linux administration and is currently intended for local network administration.

Remote Webmin access is not exposed through the public PhotoPrism endpoint.

Future work may provide private Webmin access through Tailscale while maintaining appropriate authentication and access policies.

### Samba

Samba provides local network file sharing and is restricted to the trusted home LAN.

Samba is not exposed through the public Internet.

## Security Design Principles

The networking architecture follows several security principles:

### Least Exposure

Services are not exposed externally unless there is a specific requirement.

### Separation of Services

Public-facing application access is separated from administrative services.

### Defense in Depth

Security does not depend on a single control. The design combines:

* Tailscale
* UFW
* Docker network isolation
* Reverse proxying
* Authentication
* Restricted administrative interfaces
* Backup and recovery procedures

### Minimize Port Exposure

Application containers are configured without unnecessary host port publishing.

Only services that require host-level network access are exposed.

## Future Networking Improvements

Planned improvements include:

* Tailscale Serve for private remote Webmin access
* Tailscale access policies or grants
* Webmin TOTP multi-factor authentication
* Additional network segmentation
* Improved network monitoring
* Service-specific access policies
* Periodic firewall and exposure reviews
* Documented network threat modeling
* Additional monitoring of unexpected network connections

The networking configuration will continue to evolve as additional services are added to the homelab.
