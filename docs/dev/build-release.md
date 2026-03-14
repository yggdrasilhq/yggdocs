# Build and Release

## Standard Release Command

```bash
cd ~/gh/yggdrasil
./mkconfig.sh --config ./ygg.local.toml --profile both
```

## With Guided Config

```bash
cd ~/gh/yggcli
cargo run
cd ~/gh/yggdrasil
./mkconfig.sh --config ./ygg.local.toml --profile both
```

## Publish Gate

Release only if:

1. `server` build succeeds
2. `kde` build succeeds
3. rootfs smoke tests pass for both profile artifacts
4. optional QEMU boot smoke passes when `YGG_ENABLE_QEMU_SMOKE=true`

## Optional QEMU/KVM Boot Gate

Enable VM boot checks on hosts with QEMU/KVM available:

```bash
./mkconfig.sh --config ./ygg.local.toml --profile both
```

Use `~/qemu_kvm.md` for harness prerequisites and environment validation.

## Retention

Retention is profile-specific:

- keep newest ISO from last 3 days
- keep last 2 older profile-specific ISOs
