# UPS Power Automation

Yggdrasil uses `apcupsd` to monitor the APC Back-UPS and initiate a clean shutdown
before battery exhaustion. This prevents filesystem corruption and lets the host
power back on once mains return.

## Behavior

- USB cable, local UPS monitoring.
- Shutdown triggers when either:
  - Battery charge drops below 10%.
  - Estimated runtime falls under 5 minutes.
- `ONBATTERYDELAY` of 6 seconds to avoid false triggers from brief power dips.

## Live image configuration

The build hook `config/hooks/normal/9109-apcupsd.hook.chroot` sets:

```
UPSNAME primary-host
UPSCABLE usb
UPSTYPE usb
DEVICE
ONBATTERYDELAY 6
BATTERYLEVEL 10
MINUTES 5
TIMEOUT 0
```

It also flips `/etc/default/apcupsd` to `ISCONFIGURED=yes` and enables the service
so it starts at boot.

## Verifying on a running host

```bash
systemctl status apcupsd
apcaccess status
```

## Automatic reboot on power return

`apcupsd` powers the system off cleanly and signals the UPS to cut power. The host
will boot again once AC returns, as long as the BIOS is set to "Power on after AC
loss" and the UPS resumes output automatically.

## Optional tuning

Adjust thresholds in `/etc/apcupsd/apcupsd.conf` if you want more runtime or a
faster shutdown in a longer outage.
