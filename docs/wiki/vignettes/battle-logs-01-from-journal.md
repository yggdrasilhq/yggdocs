# Battle Logs I: From Journal to Operating System

These logs are distilled from years of day-by-day notes.  
English is the base language; Bengali appears lightly where it carries tone.

## [2023-11-28] The Decision Node

The real question was not "which distro is best?"  
The real question was: "which base lets us evolve without fear?"

SmartOS had done its job, but new needs kept appearing:

- better GPU paths,
- easier kernel experimentation,
- faster service iteration.

`bujhlam`: when the platform slows learning, architecture debt starts compounding.

## [2024-03-21] Naming the System

This was the moment Yggdrasil became explicit, not accidental.

- Debian live-build became the build spine.
- ZFS, LXC, Xen expectations moved into reproducible scripts.
- Personal habits turned into shared runbooks.

Naming changed behavior: decisions now had to be repeatable, not clever.

## [2024-07-24] Boot and Recovery Experiments

Deep cycle of Xen + ZFSBootMenu + kexec testing:

- custom Xen workflows on Debian,
- resilient boot experiments,
- startup path rehearsals under failure assumptions.

Operator note: boot logic is your disaster contract in executable form.

## Migration Law: Slow Is Smooth, Smooth Is Fast

The journal repeats one operational law:

1. test in a small blast radius,
2. write the gotcha immediately,
3. migrate one subsystem at a time,
4. preserve rollback at every step.

This is less dramatic than weekend migration heroics.  
It is how systems survive.

## The Product Surface Shift

As infra stabilized, the moat became obvious:

- working recipes,
- decision stories,
- docs that help both first-timers and veterans.

`ami ei korlam` became: "here is the exact path, with checks, to do this safely."
