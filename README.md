# mac-disk-cleanup

给 Claude Code 和 Codex 用的 Mac 磁盘清理 skill。只适用于 macOS。

磁盘满了的时候，它不会按一张写死的路径清单去扫，而是跟着体积一层层往下钻，把真正占地方的东西找出来，再分成三档让你挑：A 放心删、B 逐项问、C 别动。你不明确点头，它一个文件都不删。

## 它会盯哪些容易被忽略的大户

用 AI 写代码的 Mac 上，这几类特别容易悄悄占空间：

- Claude Code 每开一个会话，就会在 `/private/tmp` 留一份 scratchpad，退出也不回收
- Go 编译缓存会一直涨
- 更新器留下的旧安装包
- AI 的 checkpoint ref 把从没 add 过的大文件钉在 `.git` 里，git 看着干净，`.git` 却可能有十几 G
- Claude Desktop 的 Cowork 沙盒镜像（skill 里写的是约 10G）

上面「约 10G」「十几 G」是 skill 里对这类文件的量级描述，不是某次清理的实测结果。

## 安装

把整个文件夹复制过去即可，目录里要有 `SKILL.md`。

### Claude Code

复制到 `~/.claude/skills/mac-disk-cleanup/`，例如：

```bash
cp -R mac-disk-cleanup ~/.claude/skills/mac-disk-cleanup
```

装好后新开一个会话，跟它说磁盘满了就行。

### Codex

按 OpenAI 目前的文档（[Build skills](https://developers.openai.com/codex/skills/)），个人 skill 放在 `~/.agents/skills/`，仓库里的 skill 放在项目的 `.agents/skills/`。个人安装可以这样：

```bash
cp -R mac-disk-cleanup ~/.agents/skills/mac-disk-cleanup
```

如果装完没出现，按文档重启一下 Codex。

## 怎么用

直接说就行，比如：

- 磁盘满了帮我看看
- 帮我腾点空间
- 清一下 mac 磁盘

它会先摸现状、分档给清单，等你点名要删哪些，再动手。

## 许可

MIT License，版权归 foamzou。
