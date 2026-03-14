# Android Agent Notes

## Key paths on device (Termux on Pixel)
- Obsidian local path: `~/storage/shared/Documents/obsidian`
- WhatsApp media: `~/storage/shared/Android/media/com.whatsapp/WhatsApp/Media/...`
- WhatsApp backups/databases: `~/storage/shared/Android/media/com.whatsapp/WhatsApp/{Backups,Databases}`
- DCIM photos/videos: `~/storage/shared/DCIM/Camera` (plus other app subdirs: IdfcFirstApp, PlantNet, pi)
- Screenshots: `~/storage/shared/Pictures/Screenshots`
- Other pics: `~/storage/shared/Pictures/PhotosEditor`, `.../Selects`, etc.
- Signal: `~/storage/shared/Signal` (for backups if enabled)
- General shared storage root: `~/storage/shared`

## Current sync jobs
- Obsidian: Termux job scheduler (Job 101) runs `android/scripts/sync-obsidian.sh` hourly on unmetered network; manual: `bash android/scripts/sync-obsidian.sh --verbose` (auto one-time `--resync` on bisync error 7). Log: `~/.local/state/ygg_client/sync-obsidian.log`.
- Boot setup: `~/.termux/boot/ygg-start-sync-jobs` calls `android/scripts/termux-boot-sync-jobs.sh` to register jobs. Widget/dynamic shortcuts installed by `android/scripts/install.sh`.

## Rclone
- Remote `smb0` expected in `~/.config/rclone/rclone.conf` (template in `android/config/rclone/rclone.conf.template`), pointing to NAS smbfs share. Obsidian path: `smb0:data/obsidian` (served from smbfs/dada/obsidian on NAS).

## yggsync (new multi-job orchestrator)
- Repo: `~/gh/yggsync` (separate Go project). Build with `GOOS=android GOARCH=arm64 CGO_ENABLED=0 go build ./cmd/yggsync`.
- Or download a release: `bash android/scripts/fetch-yggsync.sh v0.1.3` (drops into `android/bin/yggsync`).
- Re-run `android/scripts/install.sh` on device to install it into `~/.local/bin`.
- Config template: `android/config/ygg_sync.toml.template` (copy to `~/.config/ygg_sync.toml` on device).
- Primary job coverage:
  - Obsidian bisync with automatic one-time `--resync` on exit code 7.
  - WhatsApp backups + media (media prunes locally after ~12 months).
  - DCIM + Screenshots to immich/immich02 with ~31-day local retention.
  - CubeCallACR recordings with short retention.
  - androidfs catch-all with keep-latest rules for Signal/SMS/call log backups; excludes the special folders above.
- CLI: `yggsync -list`, `yggsync -jobs obsidian,dcim`, `yggsync -dry-run`.
- Safety: `retained_copy` uploads first and only prunes files confirmed on the remote (size + modtime check).
- Termux jobs (current setup):
  - Job 101 (hourly, unmetered): `android/scripts/sync-yggsync-fast.sh` (obsidian).
  - Job 102 (every 6h, unmetered, battery-not-low): `android/scripts/sync-yggsync-bulk.sh` (media/backup set).

## TODO / validation after new system lands
- Confirm `~/.config/ygg_sync.toml` paths match final NAS layout (`smbfs/dada/obsidian`, `immich/bon/DCIM`, `immich02/bon/android`, etc.).
- Ensure `termux-job-scheduler` jobs switch from `sync-obsidian.sh` to `yggsync ...` or add a new job.
- Update widgets/shortcuts to call `yggsync -jobs obsidian` and add ones for media syncs as needed.
