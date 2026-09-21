# Security

Security in ABRVN Homelab is implemented as a set of overlapping controls across the network, Linux host, administrative interfaces, containers, applications, and recovery process.

The goal is not to claim that a homelab is perfectly secure. The goal is to reduce unnecessary exposure, require stronger authentication for privileged access, keep internal services internal, preserve useful security boundaries, and validate changes instead of assuming that a control works.

> This document describes the security architecture at a sanitized level. Production addresses, credentials, private keys, account details, and sensitive configuration values are intentionally omitted.

---

## Security Model

The current design follows several principles:

- **Least necessary exposure** — a service should be reachable only where there is an operational reason.
- **Private administration** — SSH and Webmin use trusted LAN/Tailscale paths rather than the public application path.
- **Strong authentication** — administrative SSH uses public keys, and PhotoPrism accounts use multi-factor authentication.
- **Defense in depth** — firewalling, private networking, application authentication, container confinement, and backups are independent layers.
- **Internal services stay internal** — dependencies such as MariaDB do not require direct Internet exposure.
- **Security exceptions require justification** — broad container-security overrides should not remain simply because an application starts with them.
- **Validation after changes** — configuration checks, functional testing, and log review are part of hardening.
- **Recovery is a security concern** — backups and restore planning reduce the impact of destructive failures or mistakes.

---

## Trust Boundaries

The homelab separates traffic into different trust zones.

```text
                   Public / Untrusted
                          │
                          ▼
                  Tailscale Funnel
                          │
                          ▼
                        Caddy
                          │
                          ▼
                      PhotoPrism


               Trusted Administration
                          │
                       Tailscale
                     /           \
                    ▼             ▼
                   SSH          Webmin


                       Host
                        │
             ┌──────────┼──────────┐
             ▼          ▼          ▼
            UFW       Docker     Storage
                        │
                  ┌─────┴─────┐
                  ▼           ▼
             PhotoPrism    MariaDB
```

The public PhotoPrism path is not treated as a general-purpose route to the server's administrative interfaces.

---

## Host Firewall

**UFW** provides the host-level firewall layer.

The default design is restrictive: inbound access is denied unless a service has an explicit requirement.

Access is scoped by service and trusted network path. In general:

| Service | Intended access |
| --- | --- |
| SSH | Trusted LAN and Tailscale |
| Webmin | Trusted LAN and Tailscale |
| Samba | Trusted LAN |
| Tailscale | Required overlay-network connectivity |
| PhotoPrism | Intended application path through Funnel/reverse proxy |
| MariaDB | Docker-internal communication |

Exact production rules and addresses are intentionally excluded from the public repository.

The firewall is only one layer. A firewall rule does not replace secure service configuration, authentication, or application-level authorization.

---

## SSH Hardening

SSH is the primary command-line administration interface and is treated as a privileged management surface.

The current SSH posture includes:

```text
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
```

This means:

- Direct SSH login as root is disabled.
- Public-key authentication is enabled.
- Password-based SSH authentication is disabled.
- Keyboard-interactive authentication is disabled.

Administrative client devices use separate SSH keys rather than sharing one private key across every machine.

Private keys are not stored in this repository.

### Why key-only authentication?

Password authentication creates a remotely testable credential surface. Restricting SSH to authorized public keys removes normal SSH password guessing from the authentication path and allows individual client keys to be managed independently.

SSH is also restricted at the network layer rather than being intentionally exposed as a public Internet service.

---

## Tailscale and Administrative Access

Tailscale provides the private remote-access layer.

Trusted remote devices can reach approved administrative services through the Tailscale network without requiring conventional public port forwarding for those services.

This creates an important distinction:

```text
Public application access  ≠  Administrative access
```

PhotoPrism may have an intentional remote application endpoint, while SSH and Webmin remain private administrative services.

Tailscale therefore reduces the need to expose management interfaces merely because administration must sometimes occur away from the local network.

---

## Webmin Security

Webmin is a privileged administration interface and is handled differently from the public PhotoPrism application.

Current protections include:

- Access limited to trusted LAN/Tailscale paths.
- Host-firewall restrictions.
- HTTPS for the Webmin interface.
- Webmin access controls.
- Referer and CSRF protections kept enabled.
- No intentional direct public-Internet exposure.

A useful lesson from the project is that security protections should not be disabled simply to work around an access or configuration problem. If Webmin rejects a request, the cause should be investigated rather than bypassing referer or CSRF validation.

The custom Webmin Homelab Monitor is maintained as a separate module instead of modifying Webmin core files.

---

## PhotoPrism Authentication

PhotoPrism is the primary user-facing application.

Multi-factor authentication is configured for the active family accounts, adding an additional authentication factor beyond the account password.

The application is exposed only through its intended remote-access/reverse-proxy path rather than using that path to expose unrelated administrative services.

Application authentication is treated as one security layer, not a substitute for network and host controls.

---

## Docker Network Isolation

Containerized services do not all need direct host or Internet exposure.

The application design uses Docker networking so internal dependencies can communicate without publishing every service port externally.

```text
Public request
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

MariaDB is an internal dependency and does not need to become a public service.

This reduces the number of independently reachable components and makes the intended service boundary easier to understand.

---

## Container Hardening

One of the most important hardening changes in the project involved PhotoPrism's container security configuration.

An earlier configuration contained:

```yaml
security_opt:
  - seccomp:unconfined
  - apparmor:unconfined
```

These options broadly disabled two normal Docker/Linux security mechanisms for the container.

Rather than assuming the exceptions were required, they were removed incrementally and tested.

The final PhotoPrism deployment operates without that `security_opt` override. Docker therefore applies its normal seccomp filtering and the `docker-default` AppArmor profile.

Required Intel graphics device access remains available to the container.

### Validation process

The hardening change was not considered complete merely because the container started.

Validation included:

1. Checking the Compose configuration before recreation.
2. Recreating only the affected PhotoPrism service.
3. Confirming the resulting container security profile.
4. Confirming required graphics-device access remained present.
5. Testing normal photo browsing and full-resolution images.
6. Testing thumbnails and representative video playback.
7. Reviewing application logs for errors.
8. Reviewing host kernel/audit output for AppArmor denials.

No security-policy denials or functional regressions attributable to the hardening change were found during validation.

This produced an important project rule:

> **A security exception should have a demonstrated requirement. If normal protections work, keep the protections.**

---

## Reverse Proxy Boundary

Caddy provides the reverse-proxy layer for PhotoPrism.

The intended public flow is:

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
PhotoPrism
```

The reverse proxy provides a controlled application entry point. It does not turn the server into a general public gateway for SSH, Webmin, Samba, MariaDB, or other internal services.

Keeping the public path narrow reduces the number of components that must accept untrusted Internet-originated traffic.

---

## Samba

Samba is intended for trusted local file sharing.

It is not part of the public remote-access architecture and should remain scoped to the local network where required.

This is another example of assigning access based on the actual use case rather than making every useful service remotely reachable.

---

## Secrets and Repository Hygiene

The public repository documents architecture without publishing production secrets.

Material that should never be committed includes:

- Passwords
- Database credentials
- API tokens
- Authentication secrets
- SSH private keys
- TLS private-key material
- Personal photographs
- Private backups or databases

The documentation also avoids publishing unnecessary environment-specific identifiers such as production private addresses, Tailscale addresses, personal account information, and sensitive internal paths.

Sanitized examples and placeholders should be used when a configuration example is useful.

Before publishing configuration, it should be reviewed independently of whether a `.gitignore` rule exists. Ignore rules reduce mistakes but are not a substitute for checking what is actually being committed.

---

## Monitoring as a Security Control

The custom Webmin Homelab Monitor provides visibility into several security-relevant areas, including:

- UFW state
- SSH exposure
- Tailscale state
- Tailscale Funnel detection
- Docker service state
- RAID and SMART health
- Backup-job state
- Network connectivity

Monitoring does not prevent a security incident, but it can make unexpected changes or failures easier to notice.

The long-term monitoring roadmap also includes additional authentication-event visibility and alerting.

See [Monitoring](monitoring.md).

---

## Backup and Recovery

Security is not limited to preventing unauthorized access.

Accidental deletion, storage failure, broken configuration, failed upgrades, and destructive incidents can all affect availability and data integrity.

Automated backups currently protect important PhotoPrism data and supporting configuration. Disaster-recovery work remains in progress, including stronger restore validation and off-system/off-site resilience.

The project deliberately distinguishes:

```text
RAID        → storage layout / availability characteristics
Backup      → independent recovery copy
Monitoring  → detection and visibility
DR process  → documented restoration capability
```

None of these replaces the others.

---

## Change-Safety Practices

Infrastructure security can be weakened by a rushed fix just as easily as by an external threat.

Important changes therefore follow a cautious workflow where practical:

```text
Understand current state
        │
        ▼
Back up important configuration
        │
        ▼
Make one controlled change
        │
        ▼
Validate configuration
        │
        ▼
Restart/recreate only what is required
        │
        ▼
Test real functionality
        │
        ▼
Review logs / security state
        │
        ▼
Document the result
```

This approach was particularly useful during firewall, remote-access, Docker, and PhotoPrism hardening work.

---

## Defense in Depth

No single technology is treated as the security solution.

The current layers include:

```text
                   Authentication
                        ▲
                        │
Container confinement ◄─┼─► Application controls
                        │
                        ▼
                 Network isolation
                        │
                        ▼
                       UFW
                        │
                        ▼
              Tailscale / access path
                        │
                        ▼
                 Backup / recovery
```

A failure in one layer should not automatically remove every other boundary.

For example, knowing a PhotoPrism password should not by itself provide SSH access, and reaching the server over the network should not automatically expose MariaDB.

---

## What Is Intentionally Not Claimed

This project does not claim:

- That the server is immune to compromise.
- That Tailscale eliminates the need for application security.
- That Docker containers are a complete security boundary.
- That MFA protects against every account attack.
- That RAID protects data from loss.
- That a successful backup timer proves a backup can be restored.
- That monitoring prevents incidents.
- That a homelab configuration should be copied directly into another environment without review.

Security is treated as an ongoing engineering process rather than a finished checkbox.

---

## Future Security Work

Planned or potential improvements include:

- Periodic access and firewall review
- Improved authentication-event monitoring
- Failed SSH/Webmin login visibility
- Alerting for important security and service failures
- Tailscale access-policy refinement
- Additional service health checks
- More comprehensive backup integrity validation
- Formal restore drills
- Off-system/off-site backup improvements
- Storage migration away from the current non-redundant RAID design
- Continued container and dependency review
- Security review during future hardware/platform migration

These are roadmap items rather than claims about controls already deployed.

---

## Security Lessons Learned

**Public exposure should be intentional.** Remote access does not require making every service public.

**Administrative interfaces deserve a separate path.** SSH and Webmin have different risk and access requirements from PhotoPrism.

**Internal dependencies should remain internal.** A database used only by another container does not need general network exposure.

**Security defaults are valuable.** Broadly disabling seccomp or AppArmor should require a specific technical reason.

**Hardening must be tested.** A configuration that looks more secure but breaks the application is not a finished deployment; changes need functional and log validation.

**Monitoring and recovery matter.** Prevention is only one part of security. Detecting failures and recovering data are also part of operating a resilient service.

**Documentation is a control.** A rebuildable, understandable environment is easier to audit, maintain, and recover than one that depends on undocumented changes.

---

## Related Documentation

- [Architecture](architecture.md)
- [Networking](networking.md)
- [Docker](docker.md)
- [PhotoPrism](photoprism.md)
- [Monitoring](monitoring.md)
- [Operating System](operating-system.md)

A dedicated backup/disaster-recovery document will complement this security model as the recovery procedures are formalized and tested.
