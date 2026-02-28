# yggterm Operator Workflows

`yggterm` exists for the moment where infrastructure becomes too real for toy terminal habits.

You are not opening one shell. You are carrying context:

- host maintenance
- container logs
- migration tasks
- long-running fixes

A useful terminal in this context should preserve intent, not just command history.

## Practical model

1. Keep one tree branch per operational story.
2. Keep leaf sessions purpose-specific (build, incident, migration, validation).
3. Close branches only after writing a short vignette in the wiki.

## Why this matters for Yggdrasil

Yggdrasil is not a one-machine lab script; it is a living ecosystem.
The terminal should reflect that ecosystem shape.

When docs and terminal model align, onboarding becomes faster and operational mistakes drop.
