---
name: gh-smart-clone
description: Clone GitHub repos with gh smart-clone. Use when cloning, git clone, gh repo clone, forking, contributing, inspecting OSS, remotes, local git identity, grouping related repos, or creating a private copy of an external repo.
---

# gh-smart-clone

## Overview

Use `gh smart-clone` as the default GitHub checkout tool when it is installed. It encodes a path taxonomy that helps agents avoid confusing project identity, fork ownership, and write authority.

The local path should communicate what kind of work the checkout represents. Remotes should communicate where pulls and pushes go. Do not use raw `git clone` or `gh repo clone` for GitHub repositories unless the user explicitly wants that lower-level behavior, the repo is not on GitHub, or `gh-smart-clone` is unavailable.

## Scenario Guide

### First-party or Maintained Repo

Use normal mode when the repository belongs to the user, their organization, or a project they maintain as first-party work.

```sh
gh smart-clone OWNER/REPO
```

This places the checkout under the main code workspace, usually:

```text
~/Code/OWNER/REPO
```

Use this mode for ordinary owned repositories, company repositories, and repos the user treats as part of their primary workspace. It must not create forks.

### External OSS Inspection

Use `--oss` when the user wants to read, inspect, debug, run, or reference an external project without setting up a contribution fork.

```sh
gh smart-clone --oss OWNER/REPO
```

This keeps external work under the OSS boundary:

```text
~/Code/oss/OWNER/REPO
```

The `oss/` segment is a workflow boundary, not merely a tidy folder name. It tells future agents: "this is external code; be careful about authority and assumptions."

### External OSS Contribution

Use `--contribute` only when the user intends to make changes that may be submitted through a fork.

```sh
gh smart-clone --contribute OWNER/REPO
```

This mode may create GitHub state by creating or reusing a fork. It should clone the fork as `origin`, place the checkout under the upstream OSS project path, and configure `upstream` to the original project.

Expected shape:

```text
path:     ~/Code/oss/UPSTREAM_OWNER/REPO
origin:   fork owner push remote
upstream: upstream project pull remote
```

If the user passes an existing fork, treat the parent as the canonical upstream project when possible. The fork owner is a push mechanism, not the project identity.

### Private Working Copy

Use `--private` when the user wants a hidden working copy of an external project with contribution-style remotes, not a public GitHub fork.

```sh
gh smart-clone --private OWNER/REPO
```

GitHub cannot make a private fork of a public repository. This mode creates or reuses an independent private repo instead of calling `gh repo fork`. Remotes match contribution mode, but the checkout uses the private root so it does not collide with `--oss` or `--contribute`:

```text
path:     ~/Code/private/UPSTREAM_OWNER/REPO
origin:   private repo under the fork owner
upstream: upstream project pull remote
```

`--private` implies contribution setup, so `--fork-owner`, `--no-fork`, `--reconfigure`, and local git identity apply. `--contribute --private` is the same as `--private` and still uses the private root.

The private repository is not a GitHub fork, so it cannot open a pull request against upstream through the fork PR workflow. If the user intends to submit a PR, use `--contribute` instead.

### Existing Checkout

If the destination already exists, do not silently reclone over it or mutate remotes. Inspect it first:

```sh
git -C /path/to/checkout remote -v
git -C /path/to/checkout config user.name
git -C /path/to/checkout config user.email
```

Use `--contribute --reconfigure` or `--private --reconfigure` only when the user wants the existing checkout intentionally updated for contribution or private-copy remotes and optional local identity.

```sh
gh smart-clone --contribute --reconfigure OWNER/REPO
gh smart-clone --private --reconfigure OWNER/REPO
```

### Grouping Related Repos

Use `--group <name>` when the user wants several related repositories collected under one organizational folder instead of scattered directly under the owner. The group is a cosmetic directory segment placed between owner and repo; it does not change remotes, fork ownership, or local git identity.

```sh
gh smart-clone --group illo tmchow/illo-website
gh smart-clone --group illo tmchow/illo-skill
gh smart-clone --group illo tmchow/illo-characters
```

Result:

```text
~/Code/tmchow/illo/illo-website
~/Code/tmchow/illo/illo-skill
~/Code/tmchow/illo/illo-characters
```

The group folder is plain organization, not a checkout: it is never itself a git repository and never gets remotes. Grouping is explicit because it cannot be safely inferred. Do not assume a shared name prefix (`illo-*`) implies a group; only group when the user asks to. `--group` composes with every mode, so `--oss --group vendor`, `--contribute --group vendor`, and `--private --group vendor` place the grouped folder under the correct root and upstream path (`oss/` vs `private/`).

## Operating Principles

- Prefer `gh smart-clone` for GitHub clone and fork setup whenever it is installed.
- Keep project identity in the owner/repo path. For forks, the path should usually name the upstream project.
- Keep work relationship in the root. Main workspace means first-party/maintained work; `oss/` means external work; `private/` means a private working copy of external work.
- Treat fork and private-copy creation as a side effect. Only `--contribute` may create or reuse a GitHub fork. Only `--private` may create or reuse a private copy.
- Use `--oss` for inspection. Do not create forks or private copies just because a repo is external.
- Use `--private` when the user wants a hidden write remote. Use `--contribute` when they intend to submit a GitHub PR from a fork.
- Treat `--group` as cosmetic. It only organizes checkouts on disk; it never changes identity, remotes, or authority, and it is used only when the user asks to group repos.
- Verify remotes and local git identity before editing, committing, or pushing in contribution checkouts.
- Fail or ask before changing an existing checkout unless `--reconfigure` is explicit.

## Setup Philosophy

The extension is config-driven. Do not hardcode a person's account, name, email, or SSH alias in public automation.

Relevant user-level configuration:

```sh
git config --global smart-clone.prefix ~/Code
git config --global smart-clone.ossPrefix ~/Code/oss
git config --global smart-clone.privatePrefix ~/Code/private
git config --global smart-clone.forkOwner OWNER_OR_ORG
git config --global smart-clone.gitName "Example Name"
git config --global smart-clone.gitEmail person@example.com
git config --global smart-clone.sshAlias github.com-work
```

Use `--fork-owner OWNER_OR_ORG` for one-off contribution or private-copy workflows. If the fork owner differs from the authenticated `gh` user, `--contribute` treats it as an organization fork target and `--private` creates the private repo under that owner.

Use `--ssh-alias <host>` when `origin` should be SSH without relying on git config, for example `--ssh-alias github.com` or a Host alias like `--ssh-alias github.com-work`. It works in normal, `--oss`, `--contribute`, and `--private` modes. The flag overrides `smart-clone.sshAlias` when both are set.

## Verification

After contribution setup and before editing or pushing, verify:

```sh
pwd
git remote -v
git config user.name
git config user.email
gh auth status
```

Confirm these facts:

- The checkout path is under the intended first-party, `oss/`, or `private/` root.
- `origin` points to the intended fork, private copy, or owned repository.
- `upstream` points to the original project for contribution and private-copy checkouts.
- The authenticated GitHub account and local git identity match the work context.
- Existing checkout changes were made only through explicit `--reconfigure`.

## Common Pitfalls

- Cloning a contribution fork under the fork owner's path instead of the upstream project path.
- Using `--oss` when the user actually intends to contribute through a fork; use `--contribute` instead.
- Using `--private` when the user actually intends to submit a GitHub PR; use `--contribute` instead.
- Creating forks or private copies during inspection tasks. Those mutations belong only to `--contribute` and `--private`.
- Reusing a public repo or GitHub fork as a `--private` write target.
- Using the same GitHub name for a private copy and a contribution fork.
  `--contribute` and `--private` both want `<fork-owner>/<repo>`. Rename the
  private copy first; leftover GitHub rename redirects are ignored.
- Pushing with the wrong GitHub account or SSH alias.
- Mutating an existing checkout's remotes without explicit reconfiguration intent.
- Reusing a fork whose parent is not the requested upstream project.
