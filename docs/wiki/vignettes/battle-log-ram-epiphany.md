# Battle Log: RAM Epiphany and Performance Truth

Early 2024 notes captured a common ops trap:  
"System feels slow" was misread as "platform is wrong."

It was not the platform. It was memory pressure.

## What looked true

- Installed RAM looked adequate.
- CPU graphs were not screaming.
- Storage did not show obvious failure.

## What was actually true

- workload bursts pushed memory into swap behavior,
- latency spikes appeared as "random sluggishness,"
- tuning lagged behind workload reality.

`thik ache` moment: measure first, redesign later.

## Operator card

Before you migrate stacks, run this sequence:

1. sample memory + swap over real workload windows,
2. map service memory ceilings,
3. tune allocation and limits,
4. re-evaluate user-visible latency.

Migration is strategic.  
Tuning is tactical.  
Confusing the two is expensive.
