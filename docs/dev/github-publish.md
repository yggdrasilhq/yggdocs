# GitHub Publish

The public home of the ecosystem is the GitHub organization `yggdrasilhq`.

## 1) Create the repo in the org

Example:

```bash
gh repo create yggdrasilhq/yggsync --public --source ~/gh/yggsync --remote origin --push
```

Repeat the same pattern for:

- `yggdrasil`
- `yggcli`
- `yggclient`
- `yggsync`
- `yggdocs`
- `yggterm`

## 2) Verify the remote

```bash
git remote -v
git ls-remote --heads origin
```

## 3) Only after the dependency order is sane, repoint consumers

Current rule:

1. publish `yggsync` first
2. repoint `yggclient` fetch/install URLs
3. update `yggdocs`

This avoids the common mistake of shipping a public client that still points at placeholder or private dependency URLs.
