# Command Centers and SSH Keys

This page tracks machines that are allowed to control Yggdrasil hosts via root SSH keys.

## Command centers

- `pi@host-a` (existing controller)
- `pi@secondary-controller.example` (`10.10.0.240`, added as second controller)
- `root@primary-host.example` (host-level controller key)

## Persisted source of truth

- Authorized controller keys are stored in `assets/ssh/authorized_keys`.
- During ISO build, `mkconfig.sh` copies that file into:
  - `/root/.ssh/authorized_keys` in the image.

Any key that must survive rebuilds must be present in `assets/ssh/authorized_keys`.

## Updating keys

1. From a trusted admin shell, collect the pubkey:
   - `ssh pi@10.10.0.240 'cat ~/.ssh/id_ed25519.pub'`
   - `ssh root@primary-host.example 'cat /root/.ssh/id_ed25519.pub'`
2. Append missing keys to `assets/ssh/authorized_keys`.
3. Rebuild ISO with `mkconfig.sh` so future boots include the updated keyring.

## Live host one-off fix

To grant immediate access on an already-booted host (without rebuilding yet):

```bash
ssh root@primary-host.example "install -d -m 700 /root/.ssh && touch /root/.ssh/authorized_keys && chmod 600 /root/.ssh/authorized_keys"
ssh pi@10.10.0.240 'cat ~/.ssh/id_ed25519.pub' | ssh root@primary-host.example "grep -Fqx \"$(cat)\" /root/.ssh/authorized_keys || cat >> /root/.ssh/authorized_keys"
```
