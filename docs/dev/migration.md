# Migration Notes

The ecosystem is being moved from private, single-operator repos into public repos under `yggdrasilhq`.

## The important migration rule

Do not confuse “generalized” with “stripped down.”

The old private stack had real value because it encoded battle-tested infrastructure.
The public version only succeeds if that scaffolding survives, with local truth moved into config files instead of being hardcoded into the repo.

## Direction

- `yggdrasil`: ISO build spine
- `yggclient`: endpoint automation
- `yggsync`: small sync engine
- `yggcli`: operator front door
- `yggdocs`: the living manual

## Non-negotiables

1. Server and KDE builds are both first-class artifacts.
2. Docs stay ecosystem-first: `quickstart`, `wiki`, `dev`.
3. Private infrastructure values live in gitignored local config files.
4. Release discipline is part of the product, not a side process.
