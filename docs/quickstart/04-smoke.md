# 04. Run Smoke Tests

Run the complete bench before you even think about calling an image release-ready:

```bash
./tests/smoke/run.sh \
  --profile both \
  --require-artifacts \
  --with-iso-rootfs \
  --with-qemu-boot \
  --artifacts-dir ./artifacts \
  --server-iso ./artifacts/server-latest.iso \
  --kde-iso ./artifacts/kde-latest.iso
```

This validates:

- Required hooks/services in generated images.
- ZFS userland availability in both profiles.
- Boot viability in QEMU/KVM harness.

If you are inside an environment without `/dev/kvm`, still run the static and rootfs checks.
The QEMU/KVM leg is a release amplifier, not an excuse to skip the rest of the bench.

The rule is simple:

- no server image ships without smoke
- no KDE image ships without smoke
- no “I will just test it after boot” shortcuts
