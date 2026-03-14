# 01. What You Need

- Debian-based build machine with `sudo`.
- Sufficient disk space for two full ISO builds.
- `lb` (live-build), `qemu-system-x86`, `qemu-utils`, `ovmf` for optional VM smoke.
- A USB drive prepared with Ventoy for real hardware boot validation.

Optional but recommended:

- A dedicated test machine for first-boot checks.
- An APT cache/proxy, but only after the first server is alive. Do not make the first build depend on infrastructure that does not exist yet.
