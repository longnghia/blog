---
date: "2026-04-10T16:08:21+07:00"
draft: false
title: "Publishing to npm From GitHub Actions"
summary: "Automation tokens, the NPM_TOKEN secret, and a release-triggered workflow that publishes with provenance."
categories:
  - Guides
tags:
  - npm
  - github-actions
  - ci-cd
  - node
---

Publishing by hand from a laptop works right up until you're on a different machine, or you forget to run the build first, or you publish from a dirty working tree. Wiring it to GitHub releases takes about ten minutes and removes all three failure modes.

## 1. Create an npm token

The token needs to work without a human present, so it has to be an **automation** token — a normal token will fail if you have 2FA enabled on publishes, which you should.

From the CLI:

```sh
npm login
npm token create
npm token list
```

Or in the web UI: **npm → Access Tokens → Generate New Token → Automation**.

Automation tokens bypass the 2FA prompt specifically for CI. That's the point of them, and it's also why the token is worth guarding — it can publish on your behalf with nothing else.

## 2. Add it to the repo

In the GitHub repo: **Settings → Secrets and variables → Actions → New repository secret**.

Name it `NPM_TOKEN`.

## 3. The workflow

```yaml
name: Publish to npm

on:
  release:
    types: [published]

jobs:
  publish:
    runs-on: ubuntu-latest

    permissions:
      contents: read
      id-token: write

    steps:
      - name: Checkout repo
        uses: actions/checkout@v5

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          registry-url: https://registry.npmjs.org/

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test

      - name: Publish package
        run: npm publish --access public --provenance
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

## The parts that matter

**`on: release: types: [published]`** — publishing is tied to cutting a GitHub release, not to pushing a tag or merging to main. That gives you a deliberate human action as the trigger, and the release notes end up matching the version.

**`registry-url` in `setup-node`** — without this, `setup-node` doesn't write the `.npmrc` that maps `NODE_AUTH_TOKEN` to the registry, and `npm publish` fails with a 401 while your token is perfectly valid. It's the single most common reason this workflow doesn't work first time.

**`id-token: write`** — required for `--provenance`. GitHub mints a short-lived OIDC token that npm uses to attest the package was built here, from this commit. Without the permission, the publish step fails outright.

**`--provenance`** — attaches that signed attestation to the published package, so the npm page shows which repo, workflow and commit produced the tarball. Free supply-chain evidence for one flag.

**`npm ci`, not `npm install`** — installs exactly what the lockfile says and fails if `package.json` and the lockfile disagree, rather than quietly resolving something new.

**`--access public`** — needed for scoped packages (`@you/thing`), which default to private and error out if you don't have a paid account. Harmless on unscoped packages.

**`npm test` before publish** — a failing test aborts the job before anything reaches the registry. npm versions are immutable; unpublishing is restricted and disruptive. Catching it here is much cheaper than the alternative.
