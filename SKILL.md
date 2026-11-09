---
name: build-poller
description: "Install, update, or remove a macOS launchd poller that turns GitHub issues assigned to the user and labeled agent-build into draft PRs with the bundled issue-builder skill. Triggers: install build poller, update build poller, set up issue build agent, auto-build my issues, stop build poller, uninstall build poller, build poller broken."
---

# Build Poller

`build-poller.sh` runs one pass. For each open issue assigned to the `gh` user **and** labeled `agent-build`, it runs the user's agent with the bundled `skills/issue-builder/SKILL.md`. The agent builds the smallest change that solves the issue on branch `agent/issue-<n>`, tests it, pushes, and opens a **draft** PR whose description starts with `Closes #<n>`. If the issue is unclear or too big, it posts one comment on the issue instead and opens no PR.

It skips issues in repos the user cannot push to and issues that already have an open linked PR or an open PR from `agent/issue-<n>`. After a run, it builds the issue again only when the issue is edited or someone else comments; the user's own comments, including the agent's question, never trigger a run. Each run works in a fresh git worktree under `~/.local/state/build-poller/worktrees`, removed afterwards, so it never touches the user's own checkouts. A failed run is retried on the next pass. Handled keys live in `~/.local/state/build-poller/handled`.

Draft PRs are the human gate: the user reviews the draft and marks it ready. maintain-poller and other automation skip drafts until then.

Install only on macOS. When the user asks you to install the poller, do these steps:

- **Clone.** Run `git clone https://github.com/Zhachory1/build-poller ~/.local/share/build-poller`, then check out the newest release tag: `git -C ~/.local/share/build-poller checkout "$(git -C ~/.local/share/build-poller tag --sort=-v:refname | head -1)"`. If the folder already exists, follow Update instead. launchd runs the script from this path, so do not move it.
- **Check auth.** Confirm `gh auth status` succeeds, its token can push branches and open PRs in the repos where the user gets issues, and `jq` is installed. Commits, PRs, and issue comments post as this GitHub account. Pushes use `gh` as the git credential helper, set only in the poller's own clones.
- **Agent command.** Ask the user for their agent command. It must run non-interactively, take the prompt as its last argument, and be allowed to edit files and run `git`, `gh`, and the repos' build and test commands, for example `claude -p --permission-mode acceptEdits --allowedTools "Bash(git:*)" "Bash(gh:*)" "Bash(npm test:*)"`. The agent works without supervision, so prefer an allowlist over skipping permissions. Use exactly what the user gives you; do not guess.
- **Claude trust.** If the agent command runs `claude`, add `<key>PR_BUILD_TRUST_CLAUDE</key><string>1</string>` to the plist `EnvironmentVariables`. Claude Code ignores a repo's own `.claude/settings.json` (for example, its allowed test commands) until that folder is trusted, and the poller's clones are new folders. With this set, the poller marks each clone under `~/.local/state/build-poller/repos/` as trusted in `~/.claude.json` when it first uses it. Trusting a repo also enables that repo's Claude hooks, the same as opening it in Claude yourself; tell the user. Skip this step for other agents.
- **Interval.** Ask how often to poll (default 600 seconds).
- **Label.** Tell the user the poller only builds issues that are assigned to them *and* labeled `agent-build`. Adding the label is the go signal; removing it stops future runs. The label must exist in each repo (`gh label create agent-build -R <owner/repo>` if it does not). Do not create it without asking.
- **Write plist.** Write `~/Library/LaunchAgents/com.zhachory1.build-poller.plist` with this template. Replace each placeholder: `__POLLER_PATH__` with the absolute path of `~/.local/share/build-poller/build-poller.sh`, `__AGENT_CMD__` with the agent command, `__TOOL_PATH__` with the directories that contain `gh`, `jq`, and the agent CLI (`dirname "$(command -v gh)"`, etc.), and `__HOME__` with the user's home directory. Replace only the placeholders; do not change key names such as `PR_BUILD_AGENT_CMD`. Afterwards no `__` placeholder may remain. XML-escape the values (`&` → `&amp;`, `<` → `&lt;`, `"` → `&quot;`). Set `StartInterval` to the chosen interval.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0"><dict>
  <key>Label</key><string>com.zhachory1.build-poller</string>
  <key>ProgramArguments</key><array><string>/bin/bash</string><string>__POLLER_PATH__</string></array>
  <key>EnvironmentVariables</key><dict>
    <key>PATH</key><string>__TOOL_PATH__:/usr/bin:/bin</string>
    <key>PR_BUILD_AGENT_CMD</key><string>__AGENT_CMD__</string>
  </dict>
  <key>StartInterval</key><integer>600</integer>
  <key>RunAtLoad</key><true/>
  <key>StandardOutPath</key><string>__HOME__/Library/Logs/build-poller.log</string>
  <key>StandardErrorPath</key><string>__HOME__/Library/Logs/build-poller.log</string>
</dict></plist>
```

- **Load.** Run `plutil -lint` on the plist, then `launchctl bootstrap gui/$(id -u) <plist>`. If the job is already loaded (update), run `launchctl bootout gui/$(id -u)/com.zhachory1.build-poller` first. Check `~/Library/Logs/build-poller.log` after the first run.
- **Tell the agent.** Add the block below to the user's global agent instructions so their interactive agent knows the poller exists. Use the file that the user's agent reads: `~/.claude/CLAUDE.md` for Claude Code, `~/.codex/AGENTS.md` for Codex, otherwise `~/AGENTS.md`. If you are not sure, ask. Create the file if it does not exist. If the markers already exist, replace the text between them; do not add a second block. Show the user the file path and the block before you write.

```markdown
<!-- build-poller:start -->
## Local issue build poller

A launchd job (`com.zhachory1.build-poller`) runs on this Mac. For each open issue assigned to me and labeled `agent-build`, it runs `~/.local/share/build-poller/skills/issue-builder/SKILL.md` and opens a draft PR from branch `agent/issue-<n>` that closes the issue, or asks one question on the issue if it is unclear or too big.

- Do not start building an `agent-build` issue by hand unless I ask; the poller does it. Check for an `agent/issue-<n>` branch or draft PR first.
- This does not apply to a run that the poller starts. Its prompt names an issue and a worktree; do that build.
- Draft PRs from the poller need my review before I mark them ready.
- Log: `~/Library/Logs/build-poller.log`. Handled keys: `~/.local/state/build-poller/handled`.
- To stop, update, or debug the poller, follow `~/.local/share/build-poller/SKILL.md`.
<!-- build-poller:end -->
```

- **Build again.** To make the poller build an issue again without a change to the issue, delete that issue's lines from the handled file. It still skips issues with an open PR.

launchd does not start a second run of a job while the first is still running, so the poller needs no lock. The first pass builds every open issue assigned to the user with the `agent-build` label, so tell the user how many there are (`gh search issues --assignee @me --label agent-build --state open --json number --jq length`) before you load the plist.

## Update

Releases are git tags (`v0.1.0`, `v0.2.0`, ...). Install only tagged releases, never `main`. Each pass of the poller fetches tags and switches to the newest `v*` tag on its own, so script fixes apply without the user doing anything. If a new release changes `SKILL.md`, the log says `changes install steps; ask your agent to update the build poller`. If local edits block the switch, the log says `could not check out`. The steps below are for those cases, for a and for rollback.

When the user asks you to update the poller, do these steps:

- **Find versions.** Run `git -C ~/.local/share/build-poller fetch --tags`. The current version is `git -C ~/.local/share/build-poller describe --tags`. The newest is `git -C ~/.local/share/build-poller tag --sort=-v:refname | head -1`. If they match, tell the user they are up to date and stop.
- **Show changes.** Show the user the newest tag's message (`git -C ~/.local/share/build-poller tag -l --format='%(contents)' <new>`) and the files that changed (`git -C ~/.local/share/build-poller diff --stat <current> <new>`). Ask before you continue.
- **Check out.** Run `git -C ~/.local/share/build-poller checkout <new>`. If git refuses because of local changes, show them to the user and stop. Do not discard them.
- **Apply.** If only `build-poller.sh` or `skills/` changed, you are done: launchd reads the script fresh on each run. If `SKILL.md` changed, read it again and redo **Write plist**, **Load**, and **Tell the agent** from Install. Keep the user's agent command, `PATH`, and interval from the current plist; ask only about new settings.
- **Verify.** Check `~/Library/Logs/build-poller.log` after the next run.
- **Roll back or pin.** Auto-update moves back to the newest tag on the next pass. To stay on an older tag, add `PR_BUILD_UPDATED=1` to the plist `EnvironmentVariables`, check out the tag, and redo **Load**. Remove the variable to resume auto-update.

## Uninstall

When the user asks you to stop, remove, or uninstall the poller, or says it is broken, do these steps:

- **Unload.** Run `launchctl bootout gui/$(id -u)/com.zhachory1.build-poller`. This also stops a build in progress. A leftover worktree is replaced on the next run for that PR. If it fails with `3: No such process`, the job is already unloaded; continue.
- **Remove plist.** Delete `~/Library/LaunchAgents/com.zhachory1.build-poller.plist`, so the job does not load again at the next login.
- **Verify.** `launchctl print gui/$(id -u)/com.zhachory1.build-poller` must fail. If it still succeeds, report this to the user and stop.
- **Remove Claude trust.** If the plist had `PR_BUILD_TRUST_CLAUDE`, remove the trust entries the poller added: `f=~/.claude.json; d="$(cd ~/.local/state/build-poller/repos 2>/dev/null && pwd -P)"; [ -n "$d" ] && tmp="$(mktemp "$f.XXXXXX")" && jq --arg p "$d/" '.projects |= with_entries(select(.key | startswith($p) | not))' "$f" > "$tmp" && mv "$tmp" "$f"`. Do this before deleting the state folder, which the command needs. Do not change other entries.
- **Remove context.** Delete the build-poller block that **Tell the agent** added (its start marker, end marker, and every line between them) from the agent instructions file you wrote at install (`~/.claude/CLAUDE.md`, `~/.codex/AGENTS.md`, or `~/AGENTS.md`). Do not change other text in that file.
- **Ask before you delete data.** If the poller is broken, show the user the last 50 lines of `~/Library/Logs/build-poller.log` first. Then ask if they also want to delete the log, the state folder `~/.local/state/build-poller` (handled keys, cached repo clones, leftover worktrees), and the clone `~/.local/share/build-poller`. Delete only what they approve. Without the handled file, a reinstall builds each labeled issue without an open PR once more.

Branches, draft PRs, and issue comments that are already posted stay on GitHub.
