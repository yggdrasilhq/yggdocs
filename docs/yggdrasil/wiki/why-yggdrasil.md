# Why Yggdrasil

Yggdrasil exists to make a **portable, recoverable infrastructure host** that boots from USB and runs real workloads.

## What makes it different

- **ZFS-first host model** with datasets and snapshots as the operational primitive.
- **LXC-focused service model** for lightweight service isolation.
- **Bootable Debian live ISO** so recovery and replacement workflows are straightforward.
- **Two release profiles**: `server` and `kde`, both smoke-tested.
- **Operational runbooks included** for commonly deployed stacks.

## Typical use-cases

- Home-lab control plane with containerized services.
- Portable disaster-recovery host.
- Fast reprovisioning of infra workflows on new hardware.
- Experimental workstation profile with KDE plus infrastructure capabilities.

## Core USP workflows covered in docs

- Samba and network file services.
- Docker-backed app containers.
- NGINX reverse proxy/TLS fronting.
- Vaultwarden and app service patterns.
- GPU passthrough and hardware acceleration workflows.
- KVM harness boot-smoke validation before shipping ISOs.

