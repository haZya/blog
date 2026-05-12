---
title: "Why I Swapped semantic-release for release-please"
datePublished: 2026-05-12T12:20:14.006Z
cuid: cmp2lmbec00kb1qlcdqojadpx
slug: why-i-swapped-semantic-release-for-release-please
cover: https://cdn.hashnode.com/uploads/covers/698fb88f7d702cdd2e99a524/fae28e4a-b23f-4441-b8bd-c4b7f735d1e7.png
tags: automation, devops, software-engineering, github-actions, ci-cd

---

Automating the release process is one of those "set it and forget it" tasks that pays dividends for years. Traditionally, `semantic-release` was the undisputed king of this domain. It’s powerful, strictly follows Conventional Commits, and handles everything from versioning to publishing.

However, as my projects grew and my release workflows became more nuanced, I found myself looking for an alternative. That’s when I considered giving **release-please** a try.

In this article, I’ll walk you through why I made the switch, the architectural differences between these two giants, and how you can set up `release-please` in your own projects today.

## The Core Philosophy: "Release on Push" vs. "Release on Merge"

The biggest differentiator between `semantic-release` and `release-please` is *when* the release actually happens.

### semantic-release: The Continuous Delivery Purist

`semantic-release` is built on the principle of continuous delivery. Every time you push a commit to your main branch that triggers a "release" (like a `feat:` or `fix:`), it immediately creates a tag, generates a changelog, and publishes the package.

While this is great for some projects, it can be aggressive. If you push five small fixes in an hour, you get five new versions.

### release-please: The Release PR Approach

`release-please` (developed by Google) takes a different approach. Instead of releasing immediately, it maintains a **Release PR**.

1.  **The Living Changelog:** Every time you push a change to `main`, `release-please` updates a dedicated "Release PR". This PR contains the updated `CHANGELOG.md` and bumps the version in your `package.json`.
    
2.  **The "Merge to Release" Trigger:** You can see exactly what will be in the next release. When you're ready to ship, you simply merge that Release PR. *That* merge event is what triggers the actual tagging and publishing.
    

**This is the main reason I prefer it.** It gives me a "draft" state for my release. I can merge and release at any time, rather than being at the mercy of every single push.

## Why release-please Wins for Me

### 1\. Manual Version Overrides

Sometimes, the automated logic gets it wrong, or you want to force a specific version for marketing reasons. `release-please` allows you to manually set the version number in the Release PR or via configuration, which is significantly harder to do with the "black box" approach of `semantic-release`.

### 2\. Multi-Language Support

`semantic-release` is heavily rooted in the Node.js ecosystem. While plugins exist for other languages, they often feel like second-class citizens. `release-please` supports a massive array of languages out of the box, including:

*   Rust
    
*   Python
    
*   Go
    
*   Java
    
*   PHP
    
*   Ruby
    

If you’re managing a polyglot monorepo, `release-please` is a breath of fresh air.

### 3\. Automated Labels and Tags

`release-please` automatically handles GitHub labels and tags with precision. It keeps your PRs organized and your git history clean without needing a dozen different plugins.

## Deep Dive: Setting Up release-please

Let's look at a real-world configuration. In a typical setup, you'll need three things: a GitHub Action, a config file, and a manifest file.

### 1\. The GitHub Action (`.github/workflows/release-please.yml`)

This workflow runs on every push to `main`. It handles both the creation of the Release PR and the subsequent release when that PR is merged.

```yaml
name: release-please

on:
  push:
    branches:
      - main

permissions:
  contents: write
  pull-requests: write

jobs:
  release-please:
    runs-on: ubuntu-latest
    steps:
      - uses: googleapis/release-please-action@v4
        id: release
        with:
          # Use a GitHub Token with repo scope
          token: ${{ secrets.MY_RELEASE_TOKEN }}
          config-file: release-please-config.json
          manifest-file: .release-please-manifest.json

      # This job only runs if a release was actually created (PR merged)
      - name: Publish to NPM
        if: ${{ steps.release.outputs.releases_created }}
        run: |
          npm install
          npm run build
          npm publish
```

### 2\. The Configuration (`release-please-config.json`)

This file tells `release-please` which packages to track and how to format the changelog.

```json
{
  "packages": {
    ".": {
      "release-type": "node",
      "package-name": "my-awesome-lib"
    }
  },
  "changelog-sections": [
    { "type": "feat", "section": "Features" },
    { "type": "fix", "section": "Bug Fixes" },
    { "type": "chore", "section": "Miscellaneous" }
  ],
  "plugins": [
    { "type": "node-workspace" }
  ]
}
```

### 3\. The Manifest (`.release-please-manifest.json`)

This file tracks the current version of each package. `release-please` updates this automatically.

```json
{
  ".": "1.2.0"
}
```

## Tips for Success

1.  **Conventional Commits are Mandatory:** None of this works if you don't use `feat:`, `fix:`, or `feat!:`. If you're new to this, I recommend using a tool like `commitlint` to enforce it.
    
2.  **Bootstrap Your First Release:** When you first add `release-please` to an existing project, you might need to tell it where to start. You can do this by adding a `bootstrap-sha` to your config file, which points to the commit where you want the automation to begin.
    
3.  **Monorepo Power:** If you use npm/yarn/pnpm workspaces, the `node-workspace` plugin (shown in the config above) is a lifesaver. It handles cross-package dependency updates automatically when one package in the monorepo is bumped.
    

## Conclusion

Switching to `release-please` moved my release process from "anxious automation" to "controlled automation." The ability to see my changelog grow in a Release PR *before* it hits the registry has saved me from countless "patch for a patch" scenarios.

If you’re looking for a release tool that respects your workflow and supports your growth across multiple languages, give `release-please` a try.