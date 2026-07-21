# joint-renovate-test

Reproduction harness for [clientIO/joint discussion #3429](https://github.com/clientIO/joint/discussions/3429):
**do grouped `@joint/*` dependency updates converge in a single Renovate PR, or do they turn into "immortal PRs" that never merge?**

## Setup captured here

`package.json` pins five `@joint/*` runtime packages one patch behind reality
(state as of the experiment):

| package | pinned | latest on npm | update available? |
|---|---|---|---|
| `@joint/core` | 4.3.0 | 4.3.0 | no |
| `@joint/react` | 4.3.0 | **4.3.1** | **yes** |
| `@joint/layout-directed-graph` | 4.3.0 | 4.3.0 | no |
| `@joint/layout-msagl` | 4.3.0 | 4.3.0 | no |
| `@joint/decorators` | 0.4.0 | 0.4.0 | no (separate track) |

Only `@joint/react` has an update (4.3.1). The discussion's complaint is that a
lone sibling patch like this, when the scope is grouped, produces a PR that never
converges.

## Hypothesis

The immortal-PR behaviour is caused by registering `@joint/*` as a Renovate
**monorepo** (which enforces lockstep versioning), **not** by plain `groupName`
batching.

`renovate.json` here uses `groupName` **only** (no monorepo registration) plus
`rangeStrategy: "replace"`. Expected result: one clean grouped PR that bumps
`@joint/react` 4.3.0 → 4.3.1, leaves the other four untouched, and **merges /
converges normally**.

> **Note — `rangeStrategy` is scoped, not global.** It lives *inside* the
> `@joint/*` `packageRule`, so it only affects the `@joint/*` scope. Every other
> dependency in the project keeps Renovate's default per-manager strategy. Set it
> at the top level of `renovate.json` only if you want it applied project-wide.

## How to run

1. Push this repo to GitHub (already done if you're reading it there).
2. Install the **Renovate GitHub App** on the repo:
   https://github.com/apps/renovate → Configure → select this repo.
3. Renovate opens an onboarding PR. Merge it (config already present, so it's a no-op).
4. Watch for the grouped **"joint"** PR. Verify it:
   - contains only the `@joint/react` 4.3.0 → 4.3.1 bump,
   - is a single PR (batched), and
   - does not perpetually rebase/refresh (check the Dependency Dashboard issue).

## Reproducing the BROKEN config (optional)

To see the immortal PR, register the scope as a monorepo so Renovate expects
lockstep versions:

```json
{
  "extends": ["config:recommended"],
  "packageRules": [
    {
      "matchSourceUrls": ["https://github.com/clientIO/joint"],
      "groupName": "joint monorepo",
      "matchPackageNames": ["/^@joint//"]
    }
  ]
}
```

With a source-URL-based group, Renovate treats siblings as one release unit; a
lone `@joint/react` patch leaves the group version-inconsistent and the PR keeps
refreshing.
