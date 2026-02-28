# Vignette: The Undocumented Journey So Far

This page captures what the system *actually* looks like in operation, not what we hoped it looked like on a whiteboard.

A quick live audit of the running host showed the current shape clearly.

## What exists today

- One host with a ZFS-first layout.
- A large LXC fleet (30+ containers), with mixed service roles.
- Operational datasets under one pool (`zroot`) with data and container roots split by intent.
- Boot-time systemd chain for pool import and container autostart.

In short: this is no longer a prototype. It is a living machine.

## Key signals from the live system

### 1) Container reality over theory

The running fleet includes infra, app, edge, and utility containers together.
This means documentation must prioritize runbooks and recovery sequences, not abstract architecture alone.

### 2) ZFS structure is the spine

The host uses explicit dataset topology for:

- long-lived data
- LXC root filesystems
- operational state

That structure is what made repeated migrations and service growth survivable.

### 3) Boot orchestration is intentional

The service chain currently validates the intended order:

1. import zpool
2. autostart LXC
3. ensure secrets/runtime dependent services

This should stay a documentation invariant.

### 4) Local patterns became product patterns

A few patterns that started as personal habits are now core project value:

- timestamped ops journaling
- smoke-first release discipline
- profile-driven builds (server + KDE)
- recipe-based service onboarding

`bujhlam` moment: the docs are not side material, they are the product surface.

## What this means for the manual

The manual should optimize for three readers at once:

- first-time operator: "Can I do this safely?"
- experienced operator: "Can I audit/repair this quickly?"
- strategic reader: "Is this ecosystem coherent and durable?"

## Non-negotiable direction

- Keep docs ecosystem-first (`quickstart`, `wiki`, `dev`).
- Keep stories grounded in real operations.
- Keep recipes specific enough to run, generalized enough to reuse.

If Yggdrasil changes shape later, this page is the reminder: the spirit came from lived operations, not branding slides.
