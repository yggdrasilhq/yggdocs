# Battle Log: The Disk Fault That Wasn't

An HDD looked faulted during scrub cycles.  
No dramatic crash, only persistent health noise.

Root cause: cable integrity, not filesystem design.

## Triage sequence that worked

1. collect repeatable error signatures,
2. isolate physical links first,
3. re-run scrub after each hardware correction,
4. only then revisit software assumptions.

## Why this matters in Yggdrasil

ZFS gives strong signals, but humans still misclassify signals.

When an incident happens, keep hardware and software notes in one timeline.
`alada korona` (do not split them) or you lose the real story.
