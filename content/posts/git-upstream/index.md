---
date: "2026-08-25T16:32:32+07:00"
draft: false
title: "Why git checkout -b Sometimes Sets an Upstream"
summary: "Branching from origin/main configures tracking automatically; branching from a local main does not. The rule behind it is branch.autoSetupMerge."
categories:
  - Code
tags:
  - git
  - branching
---

{{< gpt >}}

Create a branch one way and `git push` just works. Create what looks like the same branch another way and Git tells you there's no upstream. The difference is one word on the command line.

```bash
git checkout -b new-feature origin/main   # upstream is set
git checkout -b new-feature main          # upstream is not set
```

## What Git actually does

Git recognizes `origin/main` as a **remote-tracking branch**, and by default has `branch.autoSetupMerge` enabled. So the first command is really two steps:

```text
create new-feature at origin/main
set new-feature's upstream to origin/main
```

Giving you:

```text
new-feature
    └── upstream: origin/main
```

The second command starts from a **local branch**, and local branches don't trigger the automatic setup:

```text
new-feature
    └── created from: main
    └── upstream: none
```

The rule Git applies is about the *kind* of start point, not the commit it resolves to:

- **local branch start point** → no automatic upstream
- **remote-tracking branch start point** → automatic tracking, subject to `branch.autoSetupMerge`

Both branches point at the same commit. Only one of them knows where to push.

## Controlling it

Turn the behaviour off globally:

```bash
git config --global branch.autoSetupMerge false
```

Or opt out for a single branch, keeping the start point:

```bash
git checkout -b new-feature --no-track origin/main
```

And to set an upstream after the fact, when you branched from a local ref and now want tracking:

```bash
git branch --set-upstream-to=origin/main new-feature
# or, on first push
git push -u origin new-feature
```

## The part worth remembering

"Based on `origin/main`" and "tracks `origin/main`" are two separate things. They happen to coincide here because of Git's branch-creation rules — not because one implies the other.
