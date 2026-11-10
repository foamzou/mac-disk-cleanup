# mac-disk-cleanup

A Mac disk cleanup skill for Claude Code and Codex. macOS only.

When the disk is full, it does not scan a fixed list of paths. It follows the largest directories down until it can tell what is actually taking space, then groups the findings into three tiers for you to pick from: A (safe to delete), B (ask item by item), and C (do not touch). It will not delete anything unless you explicitly say yes.

## Copy-paste prompts

Don't want to install by hand? Paste either prompt into your AI (Claude Code, Codex, or any agent that can read the web and run commands).

**Option 1: install it as a skill**

```text
Install this Mac disk cleanup skill for me: https://github.com/foamzou/mac-disk-cleanup
Download SKILL.md from the repo into a mac-disk-cleanup folder in your personal skills directory, then run it for me.
```

**Option 2: use it once, no install**

```text
Read SKILL.md in this repo in full, then follow it strictly to clean up my Mac disk: https://github.com/foamzou/mac-disk-cleanup
Scan only first, and list what you find in three tiers: safe to delete, ask item by item, do not touch. Do not delete any file without my explicit approval.
```

## Why it beats traditional cleanup tools

This skill is what I learned from half a year of cleaning up my own Mac. Before writing it I used cleanup tools like Tencent Lemon and Mole, but they only clean the surface. Deeper directories and other space hogs are hard for them to reach.

1. **No fixed path list.** Traditional tools only scan the folders they know about, so anything outside that list stays invisible no matter how big it is. This skill follows the size down, level by level, wherever the space actually is.
2. **Knows dev machines.** `node_modules`, build folders, the Go build cache, Docker images, Claude Code session leftovers in `/private/tmp`, and `.git` bloated by AI checkpoint refs are things traditional tools mostly don't recognize.
3. **Explains every item.** Path, size, why it's safe to delete, and what deleting it costs, so you choose, instead of one "Clean" button.
4. **Tells caches from artifacts.** Caches can be rebuilt; installer packages, `.xcarchive` files, and exported reports may be the only copy on the machine, so they are listed separately for you to decide.
5. **Won't delete recklessly.** Nothing is deleted without your OK. It checks whether a process is using something first, never uses sudo, and verifies item by item afterwards: what should be gone is gone, what should stay is still there.
6. **More honest numbers.** It reports space actually freed by the system and warns that hard links can inflate sizes, instead of double-counting APFS clones and hard links like some cleanup apps.
7. **Fixes the cause, not just the symptom.** It explains why the space comes back three days later and how to stop it, for example by moving Claude Code's temp directory somewhere that gets cleaned regularly.
8. **Handles the Claude Desktop Cowork VM.** Macs with Claude Desktop can carry a sandbox image of about 10 GB that may re-download the same day you delete it. The skill checks whether it's in use, then gives you the official config switch to turn it off and states the cost: Cowork stops working on that machine.
9. **Free and open source.** It runs on the AI you already use, with no extra cleanup app to install.

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

If it helps, a Star is very welcome. More tips are welcome as PRs too.

## License

MIT License, copyright foamzou.
