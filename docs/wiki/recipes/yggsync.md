# yggsync

`yggsync` is the small sync engine that sits underneath several `yggclient` workflows.
It is intentionally narrow: define jobs in TOML, run them through `rclone`, and make retention safer than an ad-hoc shell script usually is.

## Where it fits

Use `yggsync` when you want a handful of named sync jobs that can run the same way on a phone, a laptop, or a small workstation.

Typical patterns:

- notes mirrored both ways
- camera roll uploaded with local retention
- screenshots copied out before cleanup
- download subsets archived without dumping the whole directory

## Get it

Source:

```bash
git clone https://g.gour.top/yggdrasilhq/yggsync ~/gh/yggsync
cd ~/gh/yggsync
go build ./cmd/yggsync
install -m 0755 yggsync ~/.local/bin/yggsync
```

Or fetch a published release asset if one exists for your platform.

## First config

Start from the example file:

```bash
cp ~/gh/yggsync/ygg_sync.example.toml ~/.config/ygg_sync.toml
```

Then edit the paths and rclone remotes to match your own layout.

The example is deliberately generic. That is the point. The old private stack had the right instincts but the wrong shape for public reuse. Here the shape stays reusable and the user brings their own remote naming, storage layout, and retention policy.

## Safe first run

```bash
yggsync -config ~/.config/ygg_sync.toml -list
yggsync -config ~/.config/ygg_sync.toml -jobs notes,camera-roll -dry-run
```

Do not start with every job. Pick one small job first, confirm the remote looks right, then widen the net.

## Relationship with yggclient

`yggclient` owns the wrappers, timers, install helpers, and platform-specific glue.
`yggsync` owns the engine.

That boundary matters. It keeps the sync core small enough to read in an afternoon, and it keeps endpoint policy out of the binary.
