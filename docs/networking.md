# Networking

## Overview

The ABRVN Homelab uses a combination of local network connectivity, Tailscale, Caddy, Docker networking, and host-based firewall controls to provide reliable access to self-hosted services while minimizing unnecessary exposure.

The networking design separates **public application access** from **private administrative access**. PhotoPrism is made remotely accessible through Tailscale Funnel, while administrative services such as SSH, Webmin, and Samba remain restricted to trusted networks.

The overall design evolved from an initial attempt at traditional router-based remote access into a more controlled architecture that does not depend on exposing inbound ports directly to the Internet.

---

## Network Architecture

The general architecture is:

```text
                         Internet
                            │
                            ▼
                    Remote Access Layer
                            │
                            ▼
                     Reverse Proxy
                            │
                            ▼
                    Docker Network
                       ┌────┴────┐
                       │         │
                  PhotoPrism   MariaDB
                       │
                       ▼
                    Storage
```

Administrative access follows a separate path:

```text
Trusted LAN ───────────────► SSH
       │
       ├───────────────────► Webmin
       │
       └───────────────────► Samba

Private Remote Access
       │
       ▼
   Tailscale
       │
       └───────────────────► SSH
```

This separation prevents the public application path from becoming a general-purpose gateway into the server.

---

## Internet Connectivity

One of the major networking challenges encountered during development was the use of carrier-grade NAT (CGNAT).

With CGNAT, the address presented to the local network may not correspond to a directly reachable public IPv4 address. This prevents traditional inbound port forwarding from reliably providing remote access to services hosted inside the home network.

Rather than attempting to work around the ISP's addressing architecture, the project moved toward an overlay-network approach.

This became an important architectural decision: **remote access should not require unnecessary inbound exposure of the home network.**

---

## Tailscale

**Tailscale** provides the primary private remote-connectivity layer.

It creates an encrypted overlay network between authorized devices without requiring conventional inbound port forwarding through the home router.

This provides a useful separation between:

* Private administrative access
* Public application access
* Local network services

Tailscale is particularly useful for administration because services such as SSH and Webmin do not need to be exposed publicly simply because remote administration is required.

---

## PhotoPrism Remote Access

PhotoPrism is made publicly accessible through **Tailscale Funnel**.

The resulting access path is conceptually:

```text
Internet
   │
   ▼
Tailscale Funnel
   │
   ▼
Local HTTP Endpoint
   │
   ▼
Caddy
   │
   ▼
PhotoPrism
```

Caddy acts as the reverse-proxy layer between the incoming request and the PhotoPrism application.

This allows the public-facing portion of the infrastructure to remain relatively small while keeping the PhotoPrism application itself inside the Docker environment.

An important design consideration is that the public endpoint is intended for the **application**, not for administrative services.

---

## Caddy Reverse Proxy

Caddy provides the reverse-proxy layer for PhotoPrism.

The proxy is connected to the Docker network containing the application services and forwards requests to PhotoPrism internally.

Conceptually:

```text
Remote Request
      │
      ▼
  Tailscale
      │
      ▼
    Caddy
      │
      ▼
Docker Network
      │
      ▼
 PhotoPrism
```

Caddy also provides the TLS/reverse-proxy functionality required by the application access path.

Keeping the reverse proxy separate from the application improves service organization and makes it easier to add additional self-hosted applications in the future.

---

## Docker Network Isolation

Caddy, PhotoPrism, and MariaDB communicate through Docker networking rather than requiring every application port to be published directly on the host.

The general structure is:

```text
Docker Host
    │
    └── Proxy/Application Network
           │
           ├── Caddy
           │
           ├── PhotoPrism
           │
           └── MariaDB
```

PhotoPrism and MariaDB do not need their internal application/database ports exposed directly to the host's network interfaces.

This reduces unnecessary host-level exposure and keeps internal service communication inside the container network.

The design follows a simple principle:

> **If a service does not need to be reachable from the host network, its port should not be unnecessarily published.**

---

## Firewall

UFW provides an additional host-level access-control layer.

The firewall is used to restrict administrative and file-sharing services to trusted networks while allowing only the connectivity required by the system.

The general policy is:

| Service    | Access                                                  |
| ---------- | ------------------------------------------------------- |
| SSH        | Trusted LAN and private remote administration           |
| Webmin     | Local administrative network                            |
| Samba      | Trusted LAN                                             |
| Tailscale  | Required overlay-network connectivity                   |
| PhotoPrism | Public application path through the remote-access layer |
| MariaDB    | Docker-internal communication                           |

The exact firewall rules are intentionally not documented here because they contain environment-specific network information.

The firewall is treated as one layer of the security model rather than the only security mechanism.

---

## Administrative Access

Administrative services are intentionally separated from the public PhotoPrism access path.

### SSH

SSH is used for system administration, troubleshooting, and maintenance.

Access is restricted through the host firewall rather than being exposed as a general-purpose public service.

Private remote administration can be provided through Tailscale when required.

### Webmin

Webmin provides the primary graphical administration interface for the Linux server.

It is currently intended primarily for local network administration rather than being exposed through the public PhotoPrism endpoint.

A future iteration may provide private Webmin access through Tailscale while maintaining appropriate authentication, firewall restrictions, and access policies.

### Samba

Samba provides file-sharing functionality for trusted devices on the local network.

It is not exposed through the public Internet.

---

## Security Design Principles

The networking architecture follows several security principles.

### Least Exposure

Services are not exposed externally unless there is a specific requirement for doing so.

This reduces the number of externally reachable components and limits the potential attack surface.

### Separation of Services

Public application access is separated from administrative services.

The public PhotoPrism path does not provide access to SSH, Webmin, Samba, or the database.

### Defense in Depth

The design does not rely on a single security control.

Instead, multiple layers work together:

* Tailscale
* UFW
* Docker network isolation
* Reverse proxying
* Application authentication
* Restricted administrative interfaces
* Backup and recovery procedures

A failure or misconfiguration in one layer should not automatically expose every service on the server.

### Minimize Port Exposure

Application containers are configured without unnecessary host-level port publishing.

Only services that require host-level network access should be exposed.

---

## Design Evolution

The networking architecture changed significantly during development.

The initial approach considered traditional router port forwarding for remote access. CGNAT made that approach impractical, which led to investigating alternative connectivity methods.

The final design moved toward:

```text
Traditional Port Forwarding
          │
          ▼
      CGNAT Limitation
          │
          ▼
       Tailscale
          │
          ├──────────────► Private Administration
          │
          ▼
    Tailscale Funnel
          │
          ▼
        Caddy
          │
          ▼
      PhotoPrism
```

This was an important engineering lesson: **network architecture should account for the constraints of the underlying network rather than assuming that conventional port forwarding will always be available.**

The resulting architecture also reduced the need to expose administrative services directly to the Internet.

---

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
* Monitoring for unexpected network connections
* Additional validation of externally reachable services

Future changes will continue to follow the principle of minimizing unnecessary exposure while maintaining convenient administration.

---

## Networking Philosophy

The networking design prioritizes **controlled access over maximum accessibility**.

The goal is not to make every service reachable from everywhere. Instead, each service should have an intentional access path based on its purpose.

The resulting model is:

```text
                 ┌──────────────────────┐
                 │      Internet        │
                 └──────────┬───────────┘
                            │
                     Public Application
                            │
                            ▼
                    Tailscale Funnel
                            │
                            ▼
                         Caddy
                            │
                            ▼
                       PhotoPrism


                 ┌──────────────────────┐
                 │   Trusted Devices    │
                 └──────────┬───────────┘
                            │
                         Tailscale
                            │
                ┌───────────┼───────────┐
                ▼           ▼           ▼
              SSH        Webmin      Other
                         (Private)    Admin
```

This approach keeps the public attack surface small while preserving secure administrative access and flexible internal service communication.
