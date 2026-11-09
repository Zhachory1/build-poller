# build-poller

A macOS launchd job that turns GitHub issues into draft PRs. For each open issue assigned to you **and** labeled `agent-build`, it runs your coding agent with the bundled [`issue-builder`](skills/issue-builder/SKILL.md) skill. The agent:

- builds the smallest change that solves the issue on branch `agent/issue-<n>`, with tests;
- pushes and opens a **draft** PR whose description starts with `Closes #<n>`;
- posts one question on the issue instead, if the issue is unclear or too big for one PR.

It never opens a non-draft PR, merges, force-pushes, or requests reviewers. It skips issues that already have an open PR, and repos you cannot push to. It builds an issue again only after the issue is edited or someone else comments.

Pairs with [review-poller](https://github.com/Zhachory1/review-poller) and [maintain-poller](https://github.com/Zhachory1/maintain-poller): you review the draft, mark it ready, and assign a reviewer.

## Install

Ask your coding agent:

> Install the build poller from https://github.com/Zhachory1/build-poller by following its SKILL.md.

The agent clones this repo to `~/.local/share/build-poller`, checks `gh` and `jq`, asks you for your agent command and poll interval, loads a LaunchAgent, and adds a short note to your global agent instructions. Uninstall removes the note.

Branches, PRs, and issue comments post under your own GitHub account.

## Operate

- Build an issue: assign it to yourself and add the `agent-build` label.
- Logs: `~/Library/Logs/build-poller.log`
- Update: automatic. Each pass switches to the newest release tag. If a release changes install steps, the log tells you to ask your agent to "update the build poller".
- Stop or uninstall: ask your agent to "uninstall the build poller".
- Test: `./test.sh` (needs `jq` and `git`)

## Release (maintainer)

Installed pollers switch to a new `v*` tag on their next pass, so a pushed tag runs on every teammate's Mac within one poll interval, under their GitHub account, and can push branches and open PRs. Tag only what you would run yourself. To undo a bad release, push a newer tag with the fix.

```bash
./test.sh
git tag -a v0.2.0 -m "What changed and whether users must redo any install step"
git push origin v0.2.0
```
