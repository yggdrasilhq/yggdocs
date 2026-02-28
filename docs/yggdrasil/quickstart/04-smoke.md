# 04. Run Smoke Tests

Run the complete bench:

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
