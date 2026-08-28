# Agent Instructions

This checkout is a personal fork of `tmchow/gh-smart-clone` (`origin` = `jer-castro/gh-smart-clone`, `upstream` = `tmchow/gh-smart-clone`). It is for local use, not the release line.

- Do not bump `VERSION` or cut releases.
- If you note user-visible behavior, put it under `CHANGELOG.md` Unreleased. Do not add version headings.
- Branch personal work from `dev`. `dev` is the local integration branch. Do not retarget onto `main` unless the user asks for an upstream PR.
- Do not open PRs to `upstream` unless asked.
- Push only to `origin`, and only when the user explicitly asks.

This repository publishes `gh-smart-clone`, a script-based GitHub CLI extension, plus an agent skill that teaches future agents when to use it.

## Project Shape

- `gh-smart-clone` is the extension executable. GitHub CLI expects the repo name and root executable name to match.
- `test/run-tests.bash` is the main test suite. It uses a fake `gh` binary and real local `git` repos so tests can verify remotes and local config without touching GitHub.
- `skills/gh-smart-clone/SKILL.md` is a scenario-led agent skill. It is not a README duplicate; it should teach agents when to use normal, OSS, contribution, and private-copy modes.
- `README.md` is user-facing documentation.
- `CHANGELOG.md` records user-visible changes.

## Core Product Invariants

Preserve the three-mode model, with `--private` as a contribution-setup variant:

- Normal mode, `gh smart-clone OWNER/REPO`, is for first-party or maintained repos. It must not create forks or private copies.
- OSS mode, `gh smart-clone --oss OWNER/REPO`, is for external inspection/debugging. It must not create forks or private copies.
- Contribution mode, `gh smart-clone --contribute OWNER/REPO`, may create or reuse GitHub forks. It should place the checkout under the upstream OSS path, clone the fork as `origin`, and configure `upstream` to the original project.
- Private-copy mode, `gh smart-clone --private OWNER/REPO`, uses the same remotes as contribution mode, but creates or reuses an independent private repository instead of a GitHub fork and places the checkout under the private root (`<prefix>/private/UPSTREAM/REPO`). `--private` implies contribution setup. It must not call `gh repo fork`.

Keep path semantics clear:

- The `owner/repo` path segment represents canonical project identity.
- The `oss/` root segment represents external-work relationship.
- The `private/` root segment represents a private working copy of external work. It must not share a path with `--oss` or `--contribute`.
- Fork owner is a push mechanism, not the project identity.

Treat GitHub mutations as explicit:

- Fork creation must require `--contribute`.
- Private-copy creation must require `--private`.
- `--dry-run` and `--print-path` must never create forks or private repositories, clone, or mutate remotes.
- Existing checkout mutation must require `--reconfigure` with `--contribute` or `--private`.
- Do not silently rewrite remotes or local git identity for an existing checkout.

## Keep Surfaces in Sync

When changing CLI behavior, update all relevant surfaces in the same change:

- `gh-smart-clone` help text. Do not bump `VERSION` on this fork.
- `README.md`
- `CHANGELOG.md`
- `test/run-tests.bash`
- `skills/gh-smart-clone/SKILL.md`
- `skills/gh-smart-clone/agents/openai.yaml`, if skill-facing behavior or positioning changes

At minimum, if a feature changes how agents should choose between normal, `--oss`, `--contribute`, `--private`, or `--reconfigure`, update `SKILL.md`. The skill is the agent-facing operating model.

## Skill Guidance

`SKILL.md` should stay scenario-led. Prefer guidance about when and why to use each mode over exhaustive flag documentation.

The frontmatter `description` is the trigger surface for agents. Keep it broad enough to trigger for GitHub cloning, OSS checkouts, fork creation/reuse, remotes, and local git identity setup. Do not reduce it to a vague slogan.

Do not hardcode personal defaults in the skill. Account names, emails, and SSH aliases may appear only as clearly labeled examples.

## Testing Expectations

Run these before committing:

```sh
shellcheck gh-smart-clone test/run-tests.bash
./test/run-tests.bash
```

Validate the skill when `SKILL.md` changes. The local validator may need PyYAML:

```sh
python3 /path/to/quick_validate.py skills/gh-smart-clone
```

If your Python environment lacks PyYAML, use a temporary venv outside the repo. Do not vendor validator dependencies into this project just for local validation.

## Test Design

Keep tests non-mutating by default:

- Use the fake `gh` in `test/run-tests.bash` for fork and clone behavior.
- Use real local `git` repos inside temporary directories to verify remotes and git config.
- Avoid tests that create real forks, clone public repos, or mutate GitHub state.
- Live `gh` checks, if useful, should be dry-run or metadata-only and not required for CI.

For safety behavior, add tests for both success and refusal cases. Important refusal cases include missing required forks with `--no-fork`, wrong-parent forks, non-fork repos at the fork target, same upstream owner as fork owner, existing checkouts without `--reconfigure`, and `--private` colliding with an existing GitHub fork or public repo at the write target.

## Release and Packaging Notes

This is a script extension, so the root executable must remain executable:

```sh
chmod +x gh-smart-clone test/run-tests.bash
```

Do not treat this fork as a release: no version bumps, no GitHub releases, no PRs to `upstream` unless asked. For local installed-extension checks after the user asks to install or upgrade the fork:

```sh
gh extension upgrade smart-clone
gh smart-clone --version
gh smart-clone --contribute --dry-run OWNER/REPO
gh smart-clone --private --dry-run OWNER/REPO
```

Only use real non-dry-run `--contribute` or `--private` when the user explicitly wants to create/reuse a write repo and accepts that GitHub state may change.
