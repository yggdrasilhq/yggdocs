# KVM Harness Passthrough

This runbook mirrors the working harness model used in the dev container.

## Required passthrough devices

- `/dev/kvm`
- `/dev/vhost-net`
- `/dev/net/tun`

Validate:

```bash
ls -l /dev/kvm /dev/vhost-net /dev/net/tun
```

## Install required tooling

```bash
apt update
apt install -y qemu-system-x86 qemu-utils ovmf bridge-utils
```

Optional:

```bash
apt install -y cpu-checker virtinst libvirt-clients
```

## Verify KVM acceleration

```bash
qemu-system-x86_64 -enable-kvm -machine q35 -cpu host -nographic -serial mon:stdio -nodefaults -S
```

If KVM is available, QEMU starts without `KVM not supported` errors.
Exit with `Ctrl+a` then `x`.

## Standard UEFI test boot command

```bash
qemu-system-x86_64 \
  -enable-kvm \
  -machine q35 \
  -cpu host \
  -smp 4 \
  -m 8192 \
  -drive if=pflash,format=raw,readonly=on,file=/usr/share/OVMF/OVMF_CODE.fd \
  -drive if=pflash,format=raw,file=OVMF_VARS.fd \
  -drive file=vm.qcow2,if=virtio,format=qcow2 \
  -netdev user,id=n1,hostfwd=tcp::2222-:22 \
  -device virtio-net-pci,netdev=n1
```

## Notes

- In privileged LXC, host device ownership is inherited.
- For non-root runs, ensure user membership in the `kvm` group.
- If LXC config is regenerated, re-apply passthrough lines in container config.

