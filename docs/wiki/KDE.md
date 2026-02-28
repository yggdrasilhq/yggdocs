# KDE Plasma Desktop Environment on Yggdrasil

This document outlines the purpose, inclusion, and usage of the KDE Plasma Desktop Environment within the Yggdrasil project.

## Purpose of KDE Plasma Inclusion

The Yggdrasil project, while primarily a server-focused live OS, includes an option to build an ISO with the KDE Plasma desktop environment. This serves two main purposes:

1.  **Client System Usage:**
    *   **Temporary Live Environment:** Provide a fully functional Debian Unstable desktop environment with KDE Plasma that can be booted from USB (via Ventoy) on client systems (e.g., laptops, workstations). This is useful for system recovery, testing, or temporary work.
    *   **Debian with ZFS Installation Aid:** The KDE live environment can serve as a platform to perform a custom Debian installation, particularly for users wishing to set up Debian with ZFS on root or other complex storage configurations on their client machines. The live environment provides all necessary tools (partitioners, terminal, web browser for guides) to facilitate such an installation.

2.  **Server GUI (Optional/Temporary):**
    *   **On-Demand GUI for Server:** For server administration tasks that are significantly easier with a graphical interface (e.g., complex storage management, certain diagnostic tools, or initial setup of some services), the Yggdrasil ISO built with KDE Plasma can be booted on the server.
    *   **Ease of Use with Ventoy:** Since Yggdrasil is deployed via Ventoy, switching between a minimal server ISO and a KDE-enabled ISO is straightforward, requiring only a reboot and selection of the appropriate ISO from the Ventoy menu. This avoids permanently installing a GUI on the server if it's only needed occasionally.

## Including KDE Plasma in the Yggdrasil Build

KDE Plasma can be included in the Yggdrasil ISO by using the `--with-kde` flag when running the `mkconfig.sh` script:

```bash
./mkconfig.sh --with-kde
```

For modern AMD laptop targets (for example, ThinkBook class machines), use the ThinkBook KDE profile:

```bash
./mkconfig.sh --with-kde
```

This flag instructs the `live-build` process to:
*   Install the KDE Plasma desktop environment packages (e.g., `plasma-desktop`, `sddm` display manager, Konsole, Dolphin, and other standard KDE applications).
*   Include necessary X.org server components and drivers.
*   Configure the live system to boot into the KDE Plasma desktop session by default, likely using SDDM as the display manager.

## Key Packages Expected

When building with `--with-kde`, the following types of packages (among others) are typically included:

*   `task-kde-desktop` or `plasma-desktop` (metapackage for KDE Plasma)
*   `sddm` (Recommended display manager for Plasma)
*   `konsole` (Terminal emulator)
*   `dolphin` (File manager)
*   `plasma-nm` (Network manager applet for Plasma)
*   `xserver-xorg-core` and relevant X.org video drivers (e.g., `xserver-xorg-video-amdgpu`, `xserver-xorg-video-intel`, `xserver-xorg-video-nouveau`). For NVIDIA, if `--with-nvidia` is also used, the proprietary NVIDIA drivers would be installed and configured for X.org.
*   Common desktop applications (web browser, text editor, etc.)

## Usage Scenarios

### As a Client Live ISO

1.  Build Yggdrasil with `./mkconfig.sh --with-kde`.
2.  Copy the resulting `.iso` file to a Ventoy USB drive.
3.  Boot a client machine (laptop/desktop) from the Ventoy USB drive and select the Yggdrasil KDE ISO.
4.  The system will boot into a live KDE Plasma session.
5.  From here, you can:
    *   Use the desktop environment directly.
    *   Access ZFS tools (if included in the base Yggdrasil build) to manage ZFS pools.
    *   Use partitioning tools like `gparted` or `kde partition manager`.
    *   Perform a manual Debian installation using `debootstrap` or the standard Debian installer (if a pathway to launch it from the live system is configured or available).

### As an Optional Server GUI

1.  Build Yggdrasil with `./mkconfig.sh --with-kde`. (You might maintain two ISOs: one minimal, one with KDE).
2.  Copy the KDE-enabled `.iso` to your server's Ventoy USB drive.
3.  When a GUI is needed on the server, boot the server from the Ventoy USB and select the Yggdrasil KDE ISO.
4.  The server will boot into the KDE Plasma desktop.
5.  Perform graphical administration tasks.
6.  Once done, reboot the server and select your standard minimal Yggdrasil ISO (or other OS) from the Ventoy menu.

## Considerations

*   **ISO Size:** Including KDE Plasma will significantly increase the size of the Yggdrasil ISO image compared to a minimal server build.
*   **ISO Naming:** KDE artifacts are emitted with a `-kde` suffix (for example: `yggdrasil-YYYYMMDDHHMM-kde-amd64.hybrid.iso`).
*   **Resource Usage:** Running a full desktop environment will consume more system resources (RAM, CPU) than a minimal command-line interface. This is generally acceptable for temporary use or on client systems with adequate resources. On the server, it's understood this is for occasional use.
*   **Graphics Drivers:** Proper functioning of KDE Plasma depends on compatible graphics drivers being included and loaded. The `mkconfig.sh` script should handle the inclusion of common open-source drivers.
*   **Persistence:** By default, a live ISO environment is non-persistent. Changes made in the KDE session (e.g., installed software, saved files) will be lost on reboot unless Ventoy's persistence features are configured and used with the Yggdrasil ISO. This might be relevant for client usage if a persistent overlay is desired.

## Troubleshooting: Wayland + Missing Wi-Fi Radios

Observed issue pattern on some laptops:
1. KDE Wayland session fails to start.
2. Switching to TTY (`Ctrl`+`Alt`+`F3`) works.
3. `nmtui` shows no Wi-Fi radios/devices.

Recommended build/runbook:
1. Build KDE with `--with-kde` for AMD laptop targets.
2. Ensure firmware-heavy image profile is used (Wi-Fi 6/6E + AMD/Intel graphics firmware baked in).
3. Boot and verify with:
   - `rfkill list`
   - `nmcli radio all`
   - `ip link`
   - `dmesg | rg -i 'iwlwifi|ath|mt76|rtw|brcm|firmware|amdgpu'`
4. If Wi-Fi is absent in `ip link`, treat as firmware/driver load failure first, then review BIOS radio toggles and rfkill state.

Current KDE image defaults relevant to this issue:
* KDE profile sets hostname boot param to `yggdrasil` (default server profile remains `primary-host.example`).
* `pi` user is created by KDE build hook (`9202-set-users.hook.chroot`).
* KDE profile firmware coverage includes `firmware-linux`, `firmware-linux-nonfree`, `firmware-iwlwifi`, `firmware-atheros`, `firmware-mediatek`, `firmware-realtek`, and `firmware-brcm80211`.

This KDE Plasma option adds significant flexibility to the Yggdrasil project, broadening its utility beyond just a server OS.
