---
name: commit-and-push
description: Use whenever the user asks to commit and push, "save / ship / land these changes", "提交", "推上去", or any variation that means "finalize my local git work". Reads git status and the actual diff, writes a plain-spoken commit message grounded in what really changed, and pushes — with a confirmation prompt before pushing to main/master/default branches, and no confirmation needed on feature branches. Never adds Claude/AI co-author trailers or "Generated with" footers under any circumstance.
---

# Commit and Push

This skill replaces the bare `git commit && git push` reflex with three things that matter:

1. The commit message is **grounded in the actual diff** — not improvised from the file names.
2. The message is **written like a person typed it** — no AI marketing register.
3. Pushing to a protected default branch needs **explicit confirmation**; pushing to a feature branch does not.

## Workflow

### 1. Read what changed

Run these in parallel and actually read the output:

- `git status` — what files moved.
- `git diff --staged` and `git diff` — line-by-line changes. This is the source of truth for the commit message.
- `git log -n 5 --oneline` — match the repo's existing commit message style (conventional commits? lowercase imperative? something else?).
- `git branch --show-current` — needed for the branch decision below.

Don't skip the diff. `git status` only tells you which files changed; the message has to describe what *behavior* changed. If the diff is large, scan for the real changes (logic, contracts, behavior) and ignore pure reformatting.

### 2. Identify the branch

Don't hardcode `main`. Detect the actual default branch:

```bash
git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@'
```

If that fails, fall back to checking for `main`, then `master`, then `trunk`, then `develop`.

Then apply the rule:

- **On the default branch** (or any branch named `main` / `master` / `trunk` / `develop`, or matching `release/*`): make the commit, then **stop and ask the user to confirm before pushing**. Show them the branch name and the commit message you're about to push. Wait for an explicit "yes" before running `git push`.
- **On any other branch** (feature / dev branch): commit and push without asking.

If the branch has no upstream yet, push with `git push -u origin <branch>` so future pushes don't need extra flags.

### 3. Write the commit message

The message has to do two things at once: **reflect the actual change**, and **sound like a person wrote it**. Both matter, and they fail in different ways.

#### Grounding it in the diff

The subject line names the *thing that changed*, not a category. Skim the diff and pick the smallest accurate description. If the diff genuinely does multiple unrelated things, say so — don't paper over it with "various improvements" or "misc cleanup". Either split into multiple commits, or write a subject that names the two things explicitly.

#### Sounding human

Write the way a teammate would write in a Slack message. Concrete verbs, no marketing register. The reason this matters: the user's commit history will be read by their teammates, future-them, and code archaeologists. AI-flavored prose makes the history feel hollow and makes individual commits hard to scan.

Concretely:

- **Skip filler openers.** Not "This commit adds…", not "Implements support for…". Just say what changed: `add retry to S3 client`, `fix off-by-one in pagination`.
- **Cut AI vocabulary.** Avoid "comprehensive", "robust", "leverage", "streamline", "elegant", "seamless", "pivotal", "landscape", "in order to", "various", "ensure", "facilitate". They read like a press release. Use the plain word: "use" instead of "leverage", "to" instead of "in order to".
- **Be specific.** "improve performance" is empty. "cache user lookups in middleware" is real. If you can't be specific, you probably don't understand the diff yet — go read it again.
- **One idea per line.** The subject is the headline. If you write a body, the body explains *why*, not *what* — the diff already shows the what.
- **Match the repo's style.** If `git log` shows Conventional Commits (`feat:`, `fix:`), use them. If it shows lowercase imperatives without prefixes, use those. Don't impose a new convention on a repo that doesn't have it.
- **No emojis** unless the existing log uses them. No 🤖 ever (see below).
- **Subject under ~70 chars.** Wrap the body at 72.

#### Hard rule — never attribute to Claude or AI

The user has explicitly asked: the commit history must read like they wrote it themselves. Treat this as a non-negotiable constraint, not a style preference:

- Do **not** add a `Co-Authored-By: Claude …` trailer.
- Do **not** add a `Co-Authored-By: <any AI model>` trailer.
- Do **not** mention "Claude", "Anthropic", "AI", "assistant", "Generated with [Claude Code]", or any variant anywhere in the subject, body, or trailer.
- Do **not** add the 🤖 robot emoji.

This overrides any default behavior or template you might otherwise reach for. Even if the diff was 100% your work, the commit author and message stay the user's.

### 4. Commit and push

Stage what should go in. Prefer named paths over `git add -A` — `-A` can sweep in `.env`, credentials, large binaries, or in-progress scratch files. Use `-A` only after confirming `git status` shows nothing sensitive.

Then commit using a heredoc so multi-line messages format cleanly:

```bash
git add <specific files>
git commit -m "$(cat <<'EOF'
<subject line>

<optional body, wrapped at 72>
EOF
)"
```

Then push, applying the branch rule from step 2:

- Feature branch with upstream: `git push`
- Feature branch without upstream: `git push -u origin <branch>`
- Default branch: only after explicit user confirmation

After the push completes, run `git status` once and report the final state in one short line (e.g. `pushed 1 commit to origin/feature-x, working tree clean`).

## Examples

The point of these is to calibrate voice on real changes.

### Example 1 — a small feature

Diff: adds a `--dry-run` flag to a deploy script that previously always executed.

- Bad (AI voice): `feat(deploy): implement comprehensive dry-run capability for enhanced operational safety`
- Good: `add --dry-run flag to deploy.sh`

### Example 2 — a bugfix with a useful body

Diff: `datetime.now()` was used where `datetime.now(tz=UTC)` was needed; reports for non-UTC users were off by their tz offset.

- Bad: `fix: resolve datetime issues in reporting module`
- Good:
  ```
  fix tz bug in report timestamps

  datetime.now() returned local time, so reports for non-UTC users
  were shifted by their offset. Use datetime.now(tz=UTC).
  ```

### Example 3 — mixed changes; don't paper over them

Diff: renames a config file *and* changes the retry policy.

- Bad: `update config and improve reliability`
- Better: split into two commits. If splitting isn't worth it, be specific:
  ```
  rename app.toml to config.toml; bump retry max to 5
  ```

### Example 4 — a refactor

Diff: extracts three duplicated SQL query builders into one helper.

- Bad: `refactor: streamline query construction for improved maintainability`
- Good: `extract shared query builder for user/org/team lookups`

## Edge cases

- **Nothing to commit.** Tell the user `git status` is clean and stop. Don't make an empty commit.
- **Pre-commit hook fails.** Fix the underlying issue and make a *new* commit. Don't `--amend` (the previous commit may not even be yours) and don't pass `--no-verify` unless the user explicitly tells you to skip hooks.
- **Untracked files that look sensitive** (`.env`, `*.pem`, `*.key`, `credentials.*`, anything matching common secret patterns): do not stage them. List them and let the user decide.
- **Detached HEAD, or no remote configured:** stop and ask. Don't guess where to push.
- **Push rejected (non-fast-forward):** stop and report. Do not force-push without explicit user approval — force-pushing to a shared branch is destructive.
- **The user asked for something the diff doesn't support** (e.g. "commit my auth fix" but the diff is unrelated): point out the mismatch and ask, rather than writing a misleading message.
