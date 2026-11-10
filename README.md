# mac-disk-cleanup

给 Claude Code 和 Codex 用的 Mac 磁盘清理 skill。只适用于 macOS。

磁盘满了的时候，它不会按一张写死的路径清单去扫，而是跟着体积一层层往下钻，把真正占地方的东西找出来，再分成三档让你挑：A 放心删、B 逐项问、C 别动。你不明确点头，它一个文件都不删。

## 复制这段 prompt 就能用

不想手动装？把下面任意一段复制给你的 AI（Claude Code、Codex 等能读网页、能跑命令的都行）。

**方式一：安装成 skill，以后随时用**

```text
帮我安装这个 Mac 磁盘清理 skill：https://github.com/foamzou/mac-disk-cleanup
把仓库里的 SKILL.md 下载到你的个人 skill 目录下的 mac-disk-cleanup 文件夹，装好之后帮我运行。
```

**方式二：不安装，只用这一次**

```text
先完整读一遍这个仓库里的 SKILL.md，然后严格按它帮我清理 Mac 磁盘：https://github.com/foamzou/mac-disk-cleanup
先只扫描，按「放心删、逐项问、别动」三档给我列清单。没有我明确同意，任何文件都不要删。
```

## 比传统清理工具好在哪

这个 skill 是我半年来清理自己 Mac 沉淀下来的经验。之前用过腾讯柠檬、Mole 等清理工具，但它们只能清理表面，更深的目录和其他占用很难清到。

1. **不靠写死的路径清单。** 传统工具只扫它认识的那几类目录，名单外的再大也看不见。这个 skill 跟着体积一层层往下钻，哪里大就查哪里。
2. **懂开发机。** `node_modules`、各种 build 目录、Go 编译缓存、Docker 镜像、Claude Code 留在 `/private/tmp` 的会话残留、被 AI checkpoint ref 撑大的 `.git`，这些传统工具基本都不认识。
3. **每一项都讲清楚。** 给出路径、大小、为什么能删、删了的代价，由你来挑，而不是一个「一键清理」按钮。
4. **分得清缓存和产物。** 缓存删了能重建；打好的安装包、`.xcarchive`、导出的报告可能全机器只有一份，会单独拎出来问你。
5. **不会乱删。** 你不点头它不删；删之前先查有没有进程在用，不用 sudo，删完逐项核对：该删的删了，该留的还在。
6. **报的数字更准。** 按系统实际回收的空间算，并提醒硬链接会让体积看着偏大，不会像有的清理 App 那样把 APFS 克隆和硬链接重复计算。
7. **治根，不只清一次。** 告诉你为什么三天又涨回来，以及怎么让它别再长，比如把 Claude Code 的临时目录挪到会定期清理的地方。
8. **搞得定 Claude Desktop 的 Cowork VM。** 装了 Claude Desktop 的 Mac 上有个 10G 左右的沙盒镜像，删了当天可能又自己下回来。这个 skill 会先查它是不是还在用，再给你官方配置里的关闭开关，并说清代价：这台机器的 Cowork 就不能用了。
9. **免费开源。** 用你本来就在用的 AI，不用另外装清理软件。

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

觉得有用的话，欢迎点个 Star。有更多的技巧补充，也欢迎提 PR。

## 许可

MIT License，版权归 foamzou。
