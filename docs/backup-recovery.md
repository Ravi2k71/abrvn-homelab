# Backup and Disaster Recovery

Backup and disaster recovery in ABRVN Homelab are treated as separate from normal service availability and storage configuration.

The current environment has an operational automated backup process for important PhotoPrism data and supporting configuration. However, the disaster-recovery program is intentionally still considered **in progress** because a backup is not considered fully trustworthy until restoration procedures are documented and tested.

> This document describes the recovery design at a sanitized level. Production paths, credentials, private configuration values, and personal data are intentionally omitted.

---

## Recovery Philosophy

The project distinguishes four related but different concepts:

```text
Storage
   │
   ├── provides working capacity
   │
   ▼
RAID
   │
   ├── changes storage availability characteristics
   │
   ▼
Backup
   │
   ├── creates recovery copies
   │
   ▼
Disaster Recovery
       └── defines how services and data are restored
```

Monitoring adds another layer by helping detect failures, but it does not replace any of these.

The guiding rule is:

> **A successful backup job is evidence that a backup ran. It is not proof that the system can be restored.**

---

## Current Status

The current backup system is operational and runs automatically on a scheduled basis.

It protects important parts of the PhotoPrism deployment and supporting infrastructure, including:

- PhotoPrism media/originals
- PhotoPrism application storage
- MariaDB database data through a database dump
- Reverse-proxy configuration
- Selected Webmin-related configuration
- Backup automation definitions
- A backup manifest

The backup process is managed through systemd and is monitored through the homelab monitoring layer.

The current implementation is an important recovery foundation, but several disaster-recovery tasks remain unfinished:

- Formal restore procedure
- Full restore drill
- Boot-drive failure procedure
- RAID/storage failure procedure
- Stronger off-system/off-site protection
- Additional integrity validation

These are documented as pending work rather than presented as completed capabilities.

---

## Why RAID Is Not Backup

The current primary storage array uses Linux software RAID 0.

```text
Disk A ─┐
        ├──► RAID 0 ───► Filesystem ───► Photo Library
Disk B ─┘
```

RAID 0 combines storage capacity but provides **no redundancy**. Failure of either member can make the array unusable.

Even a future migration to RAID 1 or another redundant storage design would not eliminate the need for backups.

RAID can help with certain hardware failures depending on the RAID level. It does not independently protect against:

- Accidental deletion
- Application corruption
- Malware or destructive commands
- Filesystem corruption copied across the array
- Theft
- Fire or physical damage
- Administrative mistakes
- A failed upgrade that damages data
- Loss of the entire server

Backups therefore belong to a different failure domain from production storage.

---

## Backup Architecture

The current backup flow can be represented as:

```text
                  Production Server
                         │
          ┌──────────────┼──────────────┐
          │              │              │
          ▼              ▼              ▼
      Photo Media   Application Data   Database
          │              │              │
          └──────────────┼──────────────┘
                         │
                         ▼
                  Backup Process
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
         Configuration          Manifest
              │                     │
              └──────────┬──────────┘
                         ▼
                   Backup Set
                         │
                         ▼
                 Recovery Source
```

The objective is not to preserve a specific running container. Containers are replaceable.

The important recovery assets are the persistent data, database state, configuration, deployment definitions, and documentation required to reconstruct the service.

---

## Scheduling and Automation

The backup workflow is automated with a **systemd timer and service** rather than depending on a person remembering to run a command.

The production schedule runs weekly during a low-activity overnight window.

Automation provides several benefits:

- Consistent execution
- Service-level logging
- Clear success/failure state
- Integration with monitoring
- Reduced dependence on manual routines
- Easier troubleshooting through systemd

The timer and service definitions themselves are recovery assets because they describe how the automated backup process operates.

---

## Backup Contents

### Photo originals

The original photo library is the most important user data in the PhotoPrism environment.

Application containers can be recreated. Personal media cannot simply be regenerated after loss.

### PhotoPrism storage

PhotoPrism maintains application-generated data outside the originals collection. Protecting relevant persistent application storage makes recovery more complete.

### MariaDB

The PhotoPrism database contains application state and metadata that should be backed up in a database-aware form.

The backup workflow therefore includes a MariaDB dump rather than relying only on copying a live database directory.

### Reverse-proxy configuration

Caddy configuration is included so the application access path can be reconstructed without relying on undocumented manual configuration.

### Administration and monitoring configuration

Selected Webmin and supporting configuration is included where useful for rebuilding the management environment.

### Backup automation

The backup script and related service/timer definitions are themselves included in the recovery material.

This helps avoid a situation where data is restored but the mechanism that protects the rebuilt system has to be rediscovered from memory.

### Manifest

A manifest records information about the backup set and provides an additional reference during validation and recovery.

---

## What Is Not a Backup Target

Not every file on the server needs equal backup priority.

Examples of replaceable or reproducible components can include:

- Container images available from their upstream registries
- Operating-system packages that can be reinstalled
- Temporary files
- Rebuildable caches
- Container instances themselves

The recovery strategy prioritizes **persistent state and reproducible configuration** over copying everything indiscriminately.

This reduces backup complexity while keeping focus on the information that cannot easily be recreated.

---

## Recovery Dependencies

Restoring PhotoPrism requires more than restoring one directory.

A complete recovery may depend on:

```text
Linux host
   │
   ├── Storage / filesystems
   ├── Docker + Compose
   ├── Network configuration
   ├── Firewall / access controls
   ├── Tailscale
   │
   └── Application stack
          │
          ├── Caddy
          ├── PhotoPrism
          ├── MariaDB
          └── Persistent data
```

This is why infrastructure documentation is part of the recovery strategy.

A backup can contain the data while the documentation explains how the pieces fit together.

---

## Proposed Recovery Order

The following is the intended high-level recovery sequence. It is a **recovery design**, not yet a claim that a complete disaster-recovery drill has passed.

```text
1. Stabilize or replace failed hardware
                │
                ▼
2. Install / recover Linux host
                │
                ▼
3. Recreate storage and mount structure
                │
                ▼
4. Restore required host configuration
                │
                ▼
5. Install Docker / Compose dependencies
                │
                ▼
6. Restore deployment configuration
                │
                ▼
7. Restore PhotoPrism persistent storage
                │
                ▼
8. Restore MariaDB from database backup
                │
                ▼
9. Restore photo originals
                │
                ▼
10. Restore reverse-proxy / access configuration
                │
                ▼
11. Start services in dependency order
                │
                ▼
12. Validate local application behavior
                │
                ▼
13. Validate remote access and authentication
                │
                ▼
14. Validate monitoring and backup automation
```

The exact procedure should be refined during restore testing.

---

## Failure Scenarios

### PhotoPrism container failure

A failed application container should not require restoring the photo library from backup.

The container can normally be recreated from the deployment configuration while persistent data remains outside the container.

### MariaDB failure

Database recovery may require recreating the database service and restoring the most appropriate database dump.

This scenario needs a documented, tested restore procedure.

### Configuration damage

If a configuration change breaks PhotoPrism, Caddy, Webmin, or backup automation, a known-good configuration copy can provide a recovery reference.

Version-controlled public documentation can also help reconstruct the intended architecture, although the public repository intentionally does not contain production secrets.

### RAID member failure

The current RAID 0 design has no member redundancy. A single drive failure can affect the entire array.

Recovery may therefore require replacing storage, recreating the filesystem/storage layout, and restoring protected data from backup.

This is one of the strongest reasons for the planned migration toward redundant production storage.

### Boot-drive failure

A failed system drive may require reinstalling the operating system and rebuilding host-level configuration before application data can be restored.

A formal boot-drive recovery checklist remains a roadmap item.

### Complete host loss

Loss of the entire physical server is the scenario that most clearly demonstrates why backups should eventually exist outside the same physical failure domain.

A backup stored only with the production server can still be lost with the production server.

---

## Restore Testing

Restore testing is the largest remaining step between **having backups** and **having a validated recovery process**.

A useful restore drill should verify more than whether an archive can be opened.

It should test whether:

- Backup files can actually be read.
- The MariaDB dump can be imported.
- PhotoPrism can start against restored data.
- Original media is present and readable.
- Albums/metadata/application state behave as expected.
- Caddy can reach the application.
- Authentication still works appropriately.
- Monitoring returns to a healthy state.
- Backup automation resumes after recovery.

The first restore drill should preferably use an isolated test location or replacement environment rather than risking the working production deployment.

Results should be documented, including any missing files, undocumented dependencies, permission problems, or ordering requirements discovered during the test.

---

## Backup Validation

There are several levels of validation:

### Level 1 — Job status

Did systemd report that the backup job completed successfully?

### Level 2 — Expected contents

Does the resulting backup contain the expected files, database dump, configuration, and manifest?

### Level 3 — Integrity

Can the backup files be read and validated without corruption?

### Level 4 — Restoration

Can the backup actually recreate usable data and services?

The current environment has operational automation and monitoring, while deeper restore validation remains an active disaster-recovery task.

---

## Monitoring

The custom Webmin Homelab Monitor can surface systemd backup-job state alongside the rest of the server's health information.

This provides quick visibility into whether the scheduled backup infrastructure is operating.

However:

```text
Backup job: SUCCESS
        │
        ▼
Useful operational signal
        │
        ╳
        ▼
Not proof of successful restoration
```

Monitoring should eventually be paired with failure notifications so an unsuccessful backup does not depend on someone manually noticing it.

See [Monitoring](monitoring.md).

---

## Retention

The current automated backup process retains historical backup sets rather than keeping only the newest copy.

Retention helps protect against discovering a problem after the most recent backup has already captured the bad state.

Retention policy should balance:

- Available backup capacity
- Size of the protected data
- Frequency of backups
- How far back recovery may need to go
- Importance of the data
- Future off-site storage cost

Retention is useful, but multiple historical copies on one device still do not provide geographic or physical separation.

---

## Off-System and Off-Site Recovery

A major future improvement is to place an additional recovery copy outside the production server's primary failure domain.

The long-term design should move toward:

```text
Production Data
      │
      ├────────► Local Recovery Copy
      │
      └────────► Separate / Off-Site Copy
```

An off-system or off-site copy can improve resilience against:

- Complete server loss
- Multiple-drive failure
- Theft
- Fire
- Electrical damage
- Local filesystem corruption
- Accidental destruction of local backups

The exact implementation has not yet been finalized and should not be represented as already deployed.

---

## Credentials and Recovery

Some recovery operations depend on credentials or secrets that should **not** be stored in the public repository.

Examples can include:

- Database passwords
- Authentication secrets
- Private keys
- Service credentials
- Recovery codes

Recovery documentation should describe **what secret is required and where it is securely managed**, without embedding the secret itself in public documentation.

A technically complete backup that exposes credentials publicly would create a different security failure.

---

## Documentation as a Recovery Asset

The ABRVN Homelab repository is part of the recovery strategy even though it intentionally excludes private production configuration.

Documents such as:

- [Architecture](architecture.md)
- [Hardware](hardware.md)
- [Operating System](operating-system.md)
- [Docker](docker.md)
- [Networking](networking.md)
- [PhotoPrism](photoprism.md)
- [Monitoring](monitoring.md)
- [Security](security.md)

provide the design context required to reconstruct the environment.

The repository answers questions such as:

- What services should exist?
- Which components communicate with each other?
- Which services should be public or private?
- What security controls should be restored?
- Which data is persistent?
- Which dependencies are replaceable?

Documentation cannot replace backups, but backups without documentation can be much harder to use during a stressful failure.

---

## Disaster-Recovery Checklist

The disaster-recovery phase will be considered substantially more mature when the project has verified:

- [x] Automated scheduled backup process
- [x] Database dump included
- [x] Important PhotoPrism data included
- [x] Supporting configuration included
- [x] Backup status visible through monitoring
- [ ] Documented database restore procedure
- [ ] Documented PhotoPrism restore procedure
- [ ] Full isolated restore drill
- [ ] Boot-drive failure procedure
- [ ] RAID/storage failure procedure
- [ ] Backup integrity checks beyond job success
- [ ] Separate/off-system backup copy
- [ ] Off-site recovery copy or equivalent physical separation
- [ ] Failure notification/reporting
- [ ] Periodic restore-test schedule

The unchecked items are intentionally visible. They represent engineering work still to be completed rather than weaknesses hidden from the project documentation.

---

## Lessons Learned

**RAID and backup solve different problems.** Redundancy can improve availability, but it does not create an independent recovery copy.

**Automation is necessary but insufficient.** A scheduled backup is much better than relying on memory, but restoration still needs to be proven.

**Back up state, not disposable runtime.** Persistent media, databases, configuration, and deployment definitions matter more than preserving a specific container instance.

**Recovery documentation matters before the emergency.** Reconstructing dependencies during a failure is harder than documenting them while the system is healthy.

**Keep recovery copies in different failure domains.** A backup beside the system it protects can share the same physical disaster.

**Monitoring should report backup health.** Silent backup failure can be as dangerous as having no backup at all.

**Restore testing is part of backup engineering.** The real test of a backup is whether it can return the service and data to a usable state.

---

## Next Recovery Milestones

The next major recovery work should focus on:

1. Documenting the MariaDB restore process.
2. Performing a controlled restore test.
3. Recording permissions, ownership, and service-order requirements discovered during the test.
4. Building a boot-drive rebuild checklist.
5. Defining the replacement procedure for a failed RAID/storage array.
6. Adding an independent backup copy outside the production storage failure domain.
7. Adding reliable failure reporting.
8. Scheduling periodic restore verification.

Until those steps are completed, the backup system should be described as **operational**, while the broader disaster-recovery capability remains **in progress**.
