# Vignette: Recipe Craftsmanship

A good container recipe is not a code dump.
It is an argument for reliability.

## What makes a recipe useful

A recipe should answer four questions quickly:

1. What problem does this service solve?
2. What are the minimum dependencies?
3. How do I verify it is healthy?
4. How do I recover if it fails?

## The minimum structure

- **Purpose**: one paragraph with constraints.
- **Inputs**: datasets, ports, secrets, volumes.
- **Build/Run**: exact steps.
- **Health checks**: commands with expected outputs.
- **Failure modes**: top 2-3 likely issues.
- **Upgrade note**: what to snapshot before changes.

## Anti-patterns to avoid

- giant “just run this script” pages with no explanation
- hidden assumptions about private infra naming
- no rollback notes

## Public + private compatibility

Write recipes using placeholders (`build-host.example`, `apt-cache.example`) and document where local overrides belong.
This keeps the recipe reusable while preserving your personal infrastructure freedom.

## Final test

A recipe is done when a careful newcomer can run it,
and a seasoned operator can trust it.
