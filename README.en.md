# mac-disk-cleanup

A Mac disk cleanup skill for Claude Code and Codex. macOS only.

When the disk is full, it does not scan a fixed list of paths. It follows the largest directories down until it can tell what is actually taking space, then groups the findings into three tiers for you to pick from: A (safe to delete), B (ask item by item), and C (do not touch). It will not delete anything unless you explicitly say yes.

## Copy-paste prompts

Don't want to install by hand? Paste either prompt into your AI (Claude Code, Codex, or any agent that can read the web and run commands).

**Option 1: install it as a skill**

```text
Install this Mac disk cleanup skill for me: https://github.com/foamzou/mac-disk-cleanup
Download SKILL.md from the repo into a mac-disk-cleanup folder in your personal skills directory (Claude Code: ~/.claude/skills/mac-disk-cleanup/, Codex: ~/.agents/skills/mac-disk-cleanup/). Tell me where you installed it, and don't start cleaning yet.
```

**Option 2: use it once, no install**

```text
Read this Mac disk cleanup guide in full, then follow it strictly to clean up my disk: https://raw.githubusercontent.com/foamzou/mac-disk-cleanup/main/SKILL.md
Scan only first, and list what you find in three tiers: safe to delete, ask item by item, do not touch. Do not delete any file without my explicit approval.
```

## Hidden space hogs it looks for

On a Mac used for AI-assisted coding, these are easy to miss:

- Claude Code leaves a scratchpad under `/private/tmp` for every session and does not clean it up on exit
- The Go build cache keeps growing
- Old installer packages left behind by updaters
- AI checkpoint refs can pin large files in `.git` even if they were never `git add`ed, so the working tree looks clean while `.git` itself may be more than 10 GB
- The Claude Desktop Cowork sandbox VM image (the skill describes it as about 10 GB)

The "about 10 GB" and "more than 10 GB" figures come from the skill's own size ranges. They are not results from a particular cleanup run.

## Install

Copy the whole folder, and keep `SKILL.md` inside it.

### Claude Code

Copy it to `~/.claude/skills/mac-disk-cleanup/`, for example:

```bash
cp -R mac-disk-cleanup ~/.claude/skills/mac-disk-cleanup
```

Start a new session and ask it to look at the disk.

### Codex

Per OpenAI's current docs ([Build skills](https://developers.openai.com/codex/skills/)), personal skills go in `~/.agents/skills/`, and repo-level skills go in the project's `.agents/skills/`. For a personal install:

```bash
cp -R mac-disk-cleanup ~/.agents/skills/mac-disk-cleanup
```

If it does not show up after install, restart Codex as the docs suggest.

## Example prompts

Just say it in plain language, for example:

- 磁盘满了帮我看看
- Help me free some disk space
- Clean up my Mac disk

It will first report what it found, give you the A / B / C list, and wait for you to name what to delete.

## License

MIT License, copyright foamzou.
