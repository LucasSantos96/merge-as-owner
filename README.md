# Merge As Owner

GitHub Action that merges a pull request **authenticated as a specific
account**, instead of the default `github-actions[bot]` identity used by
`GITHUB_TOKEN` — but only when the PR's author is on an allow-list.

## Why

Some platforms treat who performed a merge as a security signal. The
concrete case this was built for: **Vercel's Hobby plan silently blocks a
deployment when the commit that lands on the production branch isn't
attributable to the project's owner account** — even if a collaborator's PR
passed every CI check. Vercel only cares about identity, not CI status.

The fix isn't "trust CI more" — it's making the merge itself carry the
owner's identity, the same way it would if the owner logged in and clicked
**Merge pull request** by hand. A PAT authenticates as its owner; the default
`GITHUB_TOKEN` always authenticates as the bot. This action swaps in the PAT,
gated behind an allow-list so it doesn't turn into "anyone's PR auto-merges
as the owner."

This applies to any situation with the same shape, not just Vercel — any
integration that keys off commit/merge identity rather than CI status.

It's a plain bash **composite action** (no `npm run build` / bundled
`dist/index.js` to go stale) — the whole implementation is in
[`action.yml`](./action.yml).

## Usage

Minimal — merge a PR into whatever branch it targets, once CI passes, if the
author is trusted:

```yaml
on:
  pull_request:
    types: [opened, synchronize, reopened]
    branches: [develop]

jobs:
  test:
    # ... your existing lint/test/build job ...

  auto-merge:
    needs: test
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: read
    steps:
      # Pin to a commit SHA, not @v1 — this action handles a privileged
      # owner token, so treat it like any other third-party action taking
      # secrets: `git ls-remote https://github.com/LucasSantos96/merge-as-owner v1`
      - uses: LucasSantos96/merge-as-owner@b95d981bf1d2515def324bfe76389bef4557d819 # v1
        with:
          pr-number: ${{ github.event.pull_request.number }}
          pr-author: ${{ github.event.pull_request.user.login }}
          allowed-authors: "LucasSantos96, other-dev-username"
          owner-token: ${{ secrets.OWNER_GITHUB_TOKEN }}
```

### Full pattern: feature → develop → main, with Vercel deploying only from owner-attributed merges

```
feature/* --PR--> develop --auto PR--> main --Vercel--> Production
              (this action)      (this action again)
```

```yaml
# .github/workflows/auto-merge-develop.yml
on:
  pull_request:
    types: [opened, synchronize, reopened]
    branches: [develop]

jobs:
  test:
    # ... lint/test/build ...

  auto-merge:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: LucasSantos96/merge-as-owner@b95d981bf1d2515def324bfe76389bef4557d819 # v1
        with:
          pr-number: ${{ github.event.pull_request.number }}
          pr-author: ${{ github.event.pull_request.user.login }}
          allowed-authors: "LucasSantos96, other-dev-username"
          owner-token: ${{ secrets.OWNER_GITHUB_TOKEN }}
```

```yaml
# .github/workflows/promote-to-main.yml
on:
  push:
    branches: [develop]

jobs:
  open-and-merge-pr:
    runs-on: ubuntu-latest
    steps:
      - name: Open promotion PR
        id: pr
        env:
          GH_TOKEN: ${{ secrets.OWNER_GITHUB_TOKEN }}
        run: |
          url=$(gh pr create --base main --head develop \
            --title "Promote develop to main" \
            --body "Automated promotion" 2>/dev/null \
            || gh pr list --base main --head develop --json url --jq '.[0].url')
          number=$(basename "$url")
          echo "number=$number" >> "$GITHUB_OUTPUT"

      - uses: LucasSantos96/merge-as-owner@b95d981bf1d2515def324bfe76389bef4557d819 # v1
        with:
          pr-number: ${{ steps.pr.outputs.number }}
          pr-author: "github-actions"          # promotion PR is opened by CI itself
          allowed-authors: "github-actions"     # trust this one PR head, not arbitrary authors
          owner-token: ${{ secrets.OWNER_GITHUB_TOKEN }}
```

The second workflow trusts the *promotion PR*, not an arbitrary author —
`develop` only advances via the first workflow's gate, so by the time code
reaches `develop → main` it has already passed the allow-list check once.

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `pr-number` | yes | — | Number of the pull request to merge |
| `pr-author` | yes | — | GitHub username of the PR author |
| `allowed-authors` | yes | — | Comma/newline separated allow-list, case-insensitive |
| `owner-token` | yes | — | Token to merge with — a fine-grained PAT scoped to this repo (`Contents: write`, `Pull requests: write`) |
| `repository` | no | `github.repository` | `owner/repo` to operate on |
| `merge-method` | no | `merge` | `merge` \| `squash` \| `rebase` |
| `commit-title` | no | GitHub default | Merge commit title override |
| `commit-message` | no | GitHub default | Merge commit message override |
| `wait-for-mergeable` | no | `5` | Poll attempts (2s apart) waiting for GitHub to compute `mergeable_state` before merging |

## Outputs

| Output | Description |
|---|---|
| `merged` | `"true"` if the merge happened, `"false"` otherwise |
| `reason` | Set when `merged=false`: `author-not-allowed` \| `not-mergeable` \| `api-error` |
| `sha` | Merge commit SHA, when `merged=true` |

## Setting up `OWNER_GITHUB_TOKEN`

1. Owner logs into their **own** GitHub account.
2. Settings → Developer settings → Personal access tokens → **Fine-grained tokens** → Generate new token.
3. Restrict **Repository access** to this repo only.
4. Permissions: **Contents: Read and write**, **Pull requests: Read and write**.
5. Owner adds it as a repo secret named `OWNER_GITHUB_TOKEN` themselves —
   the token value never needs to be shared with anyone else.

## Notes

- If `pr-author` isn't in `allowed-authors`, the action **succeeds** with
  `merged=false` — it's a no-op, not a failure, so it's safe to run
  unconditionally on every PR to the branch.
- Requires the `gh` CLI, preinstalled on `ubuntu-latest` GitHub runners.
- This only changes *who* performs the merge. It does not bypass branch
  protection rules the token's account isn't allowed to bypass (e.g.
  required reviews) — the owner's PAT merges exactly as if the owner had
  clicked the button themselves.
