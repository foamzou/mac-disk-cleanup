---
name: mac-disk-cleanup
description: "Use when the user's Mac is low on disk space or asks to clean up / free space (磁盘满了、磁盘不够、清磁盘、清一下 mac、disk full、no space、清缓存、还能删什么、腾点空间、Avail 只剩). Finds reclaimable space by drilling down by size instead of a fixed path list, groups candidates into safe / ask-first / do-not-touch tiers, and never deletes without explicit user approval. Covers dev-machine space hogs: node_modules / build / target, Claude Code scratchpad in /private/tmp, $TMPDIR leftovers, Go / npm / pnpm / pip / Docker caches, old updater packages, and .git bloated by AI checkpoint refs."
---

# Mac Disk Cleanup

## 铁律

1. **删除前必须先询问用户，等到明确同意再动手。** 用户点名（「删 X」「GO」「A 全清」）= 同意；沉默不算。
2. **给清单，不批发。** 每项标 (a) 绝对路径 (b) 大小 (c) 判断依据 (d) 删了的代价，让用户挑。
3. **回收估算要写**，并说明是 apparent size（硬链接会虚高，见 §6）。
4. **删前先 `lsof` 查占用**，尤其对 24h 内动过的目录。有进程占着就是活的，别删。见 §4。
5. **区分「缓存」和「产物」。** 缓存删了重建即可；`.xcarchive` / `.zip` / `.pkg` / 导出的报告 / 数据库备份是**产物**，可能全机器只有一份，单独列出来问。
6. **绝不用 sudo 强删**；`/System`、`/private/var/db`、root 所有的条目一律跳过。
7. **`rm` 可能被别名成进废纸篓**（`alias rm` 看一眼）。删完判回收要看 `df`，不是看 `ls`，见 §6。

---

## 1. 发现法（不是路径清单）

> **这一节是核心。** 写死的白名单必然漏东西 —— 名单外的目录再大也进不了视野。永远跟着体积下钻。

### 第 0 步：读对 Avail

```bash
df -h /System/Volumes/Data       # 真实用量看这个 volume；/ 是 sealed 系统盘，Used 没参考价值
tmutil listlocalsnapshots /      # Time Machine 本地快照，会吃空间
ls -lh /System/Volumes/VM        # swapfile，一个 1G，堆很多个就是十几 G
```

### 第 1 步：home 内，跟着体积下钻

```bash
cd ~ && du -sxh */ .[^.]*/ 2>/dev/null | sort -rh | head -45
```

**不要漏 dotdir** —— `.npm`、`.nvm`、`.cache`、`.docker`、`.gradle`、`.m2`、`.cargo` 常是 GB 级。
对**每个 >2G 的**再 `du -sxh <它>/*/ | sort -rh | head`，一直钻到能判断为止。

`du -sxh */` 会跳过 dotdir，`.git` 就此隐形。异常增长的 repo 单独看：

```bash
du -sxh <repo>/.git    # 比工作区还大就是 §3.3 的病
```

### 第 2 步：home 外

```bash
for p in /Applications /Library /opt /usr/local /private/var /Users/Shared /private/tmp; do
  printf '%s\t%s\n' "$(du -sxh "$p" 2>/dev/null | cut -f1)" "$p"
done | sort -rh
```

必查：

- **`/private/tmp`** —— Claude Code scratchpad + 手动落盘的构建目录，能到 10G+，见 §3.2
- **`$TMPDIR`**（`getconf DARWIN_USER_TEMP_DIR`）—— 脚本落盘地，见 §3.4
- **`/Applications`** —— 装了不用的大 App

### 第 3 步：项目内的构建产物

只 `du -sh` 一个总数是看不见东西的，产物藏在第 3、4 层：

```bash
cd ~ && find <你放代码的目录...> -type d \
  \( -name node_modules -o -name build -o -name dist -o -name .next -o -name target \
     -o -name .venv -o -name vendor -o -name Pods -o -name DerivedData -o -name .build \) \
  -prune -print0 2>/dev/null | xargs -0 du -sxh 2>/dev/null | sort -rh | head -35
```

`-prune` 防止钻进去重复计数。几千个条目时 `xargs` 会报参数过长，改 `while read` 循环。

### 第 4 步：大文件全局 sweep

```bash
cd ~ && find . -xdev -type f -size +400M 2>/dev/null -exec du -h {} \; | sort -rh | head -25
```

专抓 VM 磁盘镜像、`.dmg` / `.xip` 安装包、模型文件、失控的 git pack、更新器下载残渣。

### 第 5 步：拿到候选后取时间戳

```bash
for p in <候选...>; do printf '%s  %s\n' "$(stat -f '%Sm' -t '%Y-%m-%d' "$p")" "$p"; done
```

时间戳是「还在不在用」的第一手证据，比目录名可靠。

---

## 2. 分档清单

### A 档 零风险（可打包推荐，等一次 GO）

| # | 位置 | 判断 / 代价 |
| --- | --- | --- |
| A1 | 项目内已不用的 `build/`、`dist/`、`target/`、`.next/`；已删项目遗留的 `node_modules` | 删了重新 install / build 即可。`git check-ignore -v <dir>` 验证已被忽略 |
| A2 | `/private/tmp` 里已结束任务的目录 | 见 §3.2 |
| A3 | `$TMPDIR` 里的死残留 | 见 §3.4 |
| A4 | `~/Library/Caches/*` 里的各类缓存 | 常见：`Yarn`、`pip`、`Homebrew`、`golangci-lint`、`node-gyp`、`ms-playwright`、`JetBrains`、`Google`。⚠️ 对应 App 在跑先退出 |
| A5 | **Go 编译缓存** `~/Library/Caches/go-build` | 按包无限累积，**不受「任务结束就清」约束**，十几天能长到十几 G。`go clean -cache` 一条命令，代价只是下次编译慢一次。跑过 Go 每轮都查 |
| A6 | `~/.npm`（`_cacache` + `_npx`） | `npm cache clean --force`。先 grep `~/.zsh_history` 确认没在依赖 `_npx` 里的东西 |
| A7 | pnpm store | `pnpm store prune`，只清没人引用的包 |
| A8 | **更新器旧版本安装包** | 旧的 0 容忍，**最新的待安装版本要留**，见 §2.1 |
| A9 | `brew cleanup --prune=all` | 清旧版本和下载缓存 |
| A10 | App 已卸载、扩展目录还在 | 如已卸载编辑器遗留的 `~/.<编辑器名>` 扩展目录。判据：`ls /Applications \| grep -i <名字>` 为空 = 纯孤儿 |

### 2.1 更新包：旧的清，最新的留

判据只有一条：**这个包是「已经装完 / 已经过期的旧版本」吗？**

| 删 | 留 |
| --- | --- |
| 同目录里有更新版本躺在旁边的旧版本 | 版本号最高的那一个 —— 它等着被装 |
| `*/ShipIt/update.<旧hash>`、`*/Updates/<旧版本>-<hash>` | 当前正在被处理的（`pgrep -fl ShipIt`） |
| `$TMPDIR/*ShipIt.*` 跑完留下的 | `*Updater/Current/`、`*Updater/apps/`（更新器引擎本体） |

**别一看到 `.pkg` / `update.zip` 就整目录删。** 先看目录里有几个、版本号排一遍，只删次新的和更老的。

扫法：

```bash
find ~/Library/Caches ~/Library/Application\ Support -maxdepth 2 \
  \( -iname "*updater*" -o -iname "*ShipIt*" -o -iname "*Update*" \) 2>/dev/null |
  while read -r p; do printf '%s\t%s\n' "$(du -sxh "$p" 2>/dev/null | cut -f1)" "$p"; done | sort -rh
T=$(getconf DARWIN_USER_TEMP_DIR); du -sxh "$T"*ShipIt* 2>/dev/null
```

先查有没有更新正在进行：`pgrep -fl ShipIt`，在跑的暂存目录不能删。

### B 档 需逐项问

| # | 位置 | 要问的 |
| --- | --- | --- |
| B1 | Docker / Rancher Desktop / Colima 的 VM 磁盘镜像（`~/Library/Containers/com.docker.docker`、`~/Library/Application Support/rancher-desktop/lima/*/diffdisk`） | 动态膨胀，常 10G+。里面有本地镜像和 volume 数据。优先用 `docker system prune`，别直接删文件 |
| B2 | `~/Library/Application Support/Claude/vm_bundles/claudevm.bundle` | Claude Desktop Cowork 沙盒镜像（`rootfs.img` 约 10G + `rootfs.img.zst` 约 1G）。Cowork 是懒启动的，只切标签页不拉 VM，真发消息才重建。删前 `pgrep -fl claudevm` 无进程 + `rootfs.img` 的 mtime 没变。**⚠️ 光删没用：可能当天自己重下**（`rootfs.img.tmp` 一路涨回 10G）。复查：`ls -lh .../claudevm.bundle/` 看有没有 `.tmp` / `.partial` 在长 |
| B2b | 不想再要 Cowork（B2 的根治） | 在 `~/Library/Application Support/Claude/claude_desktop_config.json` 的 `preferences` 里加 `"secureVmFeaturesEnabled": false` 并重启 Claude Desktop。这是 Claude Desktop 官方企业配置项（默认 true，含义「Enable Cowork access in desktop」，见 support.claude.com 的 Enterprise configuration for Claude Desktop），**代价是这台机器上 Cowork 整个不可用**，要用 Cowork 就别设 |
| B3 | `~/.local/share/opencode`、`~/.codex/sessions`、`~/.claude/projects` | Agent 对话历史，删了没法回溯 |
| B4 | `~/.nvm/versions`、`~/.local/share/mise/installs`、`~/.gvm` | ⚠️ **先确认 `claude` / `codex` 等 CLI 装在哪个版本下**，删那个版本会连带删掉它 |
| B5 | `~/go/pkg/mod` | `go clean -modcache`，代价是下次重新下载 |
| B6 | `~/.gradle`、`~/.m2`、`~/.cargo/registry` | 重新拉依赖，慢一次 |
| B7 | `~/Library/Application Support/Google/Chrome/Snapshots` 等浏览器旧 profile | 有书签/历史，收益小风险非零 |
| B8 | 孤儿目录 | 在 home 误跑 `npm i` 留下的 `~/node_modules`、`~/dist`、`/Users/Deleted Users`，按 mtime 判 |
| B9 | repo 的 `.git/objects` 被 AI checkpoint ref 撑爆 | 要删 ref + 跑 gc，见 §3.3。**主分支干净就绝不改历史** |
| B10 | 本机模型缓存（TTS / 本地 LLM / `~/.cache/huggingface`） | 删了下次用要重下，大的几 G |
| B11 | `/usr/local` 老树（Intel → Apple Silicon 迁移遗留） | 见 §5 |

### C 档 别动

- `/private/var/folders` 的 **root 那部分**（用户自己的 `T/`、`C/` 例外，见 §3.4）
- `/System/*`、`/private/var/db/*`
- 微信 / 企业微信 / 飞书 等 IM 的数据目录（`~/Library/Containers/com.tencent.*` 等）—— 在 App 内清
- `~/Library/Application Support/Claude` 除 `vm_bundles` 外的子目录
- 公司或学校装的终端安全软件（如 Sophos、CrowdStrike）的日志和缓存（root 所有，且是活的 agent，不动）

---

## 3. 几个「隐形大户」的机制

### 3.1 项目内每个任务一份、从不回收的构建目录

排增量编译的坑时为了不吃上次缓存，会换一个新的输出目录（`-dd`、`build-xxx`、`out-test`）。名字跟着当时的活儿走，编完没人回来删。**每个项目按 mtime 留最新一份，其余删。**

必须挑出来单独问的例外：`*.xcarchive`（带 dSYM）、`*.zip` / `*.pkg`（打好的分发包）、benchmark / 测试报告 —— 这些是产物，不是缓存。

### 3.2 `/private/tmp` —— Claude Code 自己在往里写

**这是 Claude Code 的内建行为**：每开一个会话就在 `/private/tmp/claude-<uid>/<sessionId>/scratchpad` 建一层，退出不回收。跑并行 agent 的会话单个能留 **数 G**。macOS 不会自动清它，几个月前的条目还在。

`/private/tmp` 里另一半是手动落盘的构建目录、clone 下来的 repo、临时脚本产物。

**治根**：Claude Code 支持 `CLAUDE_CODE_TMPDIR`，指到一个会定期清的位置。

### 3.3 `.git/objects` 被 AI 工具的 checkpoint ref 撑爆

**只要往 repo 目录里放过一个大文件，Codex / Claude 的 turn 快照 ref 就可能把它永久钉在 `.git/objects` 里 —— 即使它从没被 `git add`、也从没进过任何分支。**

三个「看起来干净」的假象：`git ls-files` 说未跟踪；`git log --all -- <path>` 说没有 commit；`du -sh .git` 却有十几 G（挂在 `refs/codex/turn-diffs/*` 上）。

**定位（三步，别猜）**：

```bash
# 1) 找出 >100MB 的 blob，连路径一起
git rev-list --objects --all | git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' \
  | awk '$1=="blob" && $3>104857600 {printf "%.2f GB  %s  %s\n", $3/1073741824, $2, $4}' | sort -rn

# 2) 逐个 ref 测谁能到达它 —— 这一步决定修法
git for-each-ref --format='%(refname)' | while read -r r; do
  n=$(git rev-list --objects "$r" 2>/dev/null | grep -cE "^(<blob1>|<blob2>) ")
  [ "$n" -gt 0 ] && echo "$n  $r"
done

# 3) 确认主分支干净（干净就绝对不要改历史）
git rev-list --objects refs/heads/<主分支> | git cat-file --batch-check='%(objecttype) %(objectsize) %(rest)' \
  | awk '$1=="blob" && $2>20971520'
```

**修法取决于第 2 步**：

- **只挂在 `refs/codex/*` / `refs/original/*` / stash 这类脚手架 ref 上** → 删那几个 ref + `git reflog expire --expire=now --expire-unreachable=now --all` + `git gc --prune=now`。主分支一个 commit 都不动。
- **真的进了分支历史** → 才需要 `git filter-repo --invert-paths`（会移除 origin、要 force push，协作 repo 上有代价，先跟团队商量）。

**别一上来就 `filter-repo`**，先做第 2、3 步。

顺带信号：「push 一直失败 + `.git` 异常大」基本就是这个病（GitHub 单文件上限 100MB）。

### 3.4 `$TMPDIR` —— 脚本自己的落盘地

```bash
T=$(getconf DARWIN_USER_TEMP_DIR)   # /var/folders/<xx>/<hash>/T/
du -sxh "$T"* 2>/dev/null | sort -rh | head -15
```

「`/private/var/folders` 别手动删」**只对 root 那部分成立**；`T/`（temp）和 `C/`（cache）是当前用户自己的。

常见构成：`pip-unpack-*` / `pip-metadata-*`（pip 解压残留）、`go-build[0-9]*`（go 编译临时目录，进程非正常退出就留下）、`tmp.*`（`mktemp -d` 的脚本从不清）、`Xcode*` 流水线残留。

**例外，要留 / 要验**：`node-compile-cache`（有用）；`CFNetworkDownload_*.tmp`（**可能正在下载**，必须 lsof 验）；各 App 的模型缓存（App 在用）。

**只清已识别类别，不要 `rm -rf $TMPDIR/*`** —— 里面混着运行中 App 的 socket 和锁文件。

---

## 4. 判「还在不在用」

**判据是「任务结束了没有」，不是「几天前」。** 机器刚重启过时，`/private/tmp` 可能整个落在「3 天内」，按天数一刀切切不下去；而其中没有一个还在用。

按顺序：

```bash
# 1) 有没有进程占着（最硬的证据）
lsof +D <目录> 2>/dev/null | tail -n +2 | wc -l
lsof <文件> 2>/dev/null | head -3

# 2) 还在不在长（下载中的文件、正在编译的目录）
stat -f '%z' <文件>; sleep 3; stat -f '%z' <文件>

# 3) 递归取内部最新 mtime，不要只看顶层目录的 mtime
find <目录> -type f -exec stat -f '%m' {} + 2>/dev/null | sort -rn | head -1
```

用户说「不要删最近 N 小时改过的」时，必须用第 3 条的递归 mtime，只 `stat` 顶层会误删正在重编的目录。

---

## 5. `/usr/local` 老树（只在 Apple Silicon 且迁过 brew 的机器上）

Intel → Apple Silicon 迁移后，`/usr/local/Cellar`、`/usr/local/Homebrew` 可能留着一整棵 x86 老树。

**先验再说**：

```bash
arch -x86_64 /usr/bin/true       # Bad CPU type = 没装 Rosetta，老树里的 x86 二进制都跑不起来
du -sxh /usr/local/Cellar /usr/local/Homebrew /usr/local/lib 2>/dev/null
brew list --formula | sort > /tmp/new.txt
/usr/local/bin/brew list --formula 2>/dev/null | sort > /tmp/old.txt   # 老 brew 跑不起来就读 Cellar 目录名
comm -12 /tmp/new.txt /tmp/old.txt   # 两边都有 → 旧的冗余
comm -13 /tmp/new.txt /tmp/old.txt   # 老树独有 → 可能有跨架构工具，逐个看
```

**整棵树不能盲删** —— 里面混着跨架构能用的东西：Python 实现的 CLI（如 `google-cloud-sdk`）、`adb`、各种 `#!/bin/sh` 脚本。逐个 `<命令> --version` 验过再决定。

**别为了腾空间劝人装 Rosetta**，那只是让没人用的 x86 工具重新可用，纯反向。

---

## 6. 执行时的坑

- **APFS Avail 释放延迟**：`rm -rf` 后 `df` 常几分钟才追上。`sync; sleep 5; df -h /System/Volumes/Data`。还没变就别重复删，`ls` 确认目录没了就是成功。
- **`du` 跨目录重复计数硬链接**：Go 模块 cache、CocoaPods、pnpm / Bun cache 常在多目录间硬链接。分别 `du -sh A B` 相加会重复算。要提前说明：报的是 apparent size，实际释放可能远小于此。
- **大目录删除慢**：上百个 `node_modules` / DerivedData 连删能超 2 分钟。用 `&` 并行 + `wait`，或后台跑。
- **`find -newermt` 在 macOS 上不可靠**：算时间线用 `CUT=$(date -v-3d +%s)` 加 `stat -f '%m'` 手工比。
- **Permission denied 是正常的**：`/private/tmp/powerlog` 这类 root 条目跳过即可，不要 sudo。
- **`rm` 被别名成进废纸篓时**（`alias rm`、`safe-rm`、`trash` 之类）：删完磁盘并没释放。核对 `du -sxh ~/.Trash`；要真释放走 `/bin/rm -rf ~/.Trash/<条目>`。反过来，磁盘紧时 macOS 会自己清废纸篓，「Trash 是空的」不等于「已真删」。**判回收只看 `df`。**
- **第三方清理 App（Tencent Lemon、CleanMyMac 等）报的体积偏高**：APFS 克隆/硬链接被按逻辑大小重复计算。以 `du` 为准。

---

## 7. 治根（不然一周后原样长回来）

清掉几十 G 后三天涨回来是常态 —— 清理速度追不上产生速度。

| 手段 | 做法 |
| --- | --- |
| 构建产物谁造谁收 | 让 agent / 脚本把落盘统一到 `/private/tmp/dd/<项目>/<任务>/`，**任务成功就地删，失败留着排查**，别散在 `/private/tmp` 根下 |
| 挪走 Claude scratchpad | `CLAUDE_CODE_TMPDIR` 指到会定期清的位置 |
| 脚本自己收尾 | 用 `mktemp -d` 的脚本加 `trap 'rm -rf "$TMP"' EXIT` |
| 每次都换新构建目录是对的 | 改过工程配置 / 生成代码后旧缓存会残留过期模块，编出来的结果不可信。问题不在「新建」，在「编完不删」。别劝人复用固定路径 |
| 扫描告警 | 定时 `df -h /System/Volumes/Data`，Avail 跌破阈值发通知。**只告警，下刀仍要用户在场** |

**不要写 launchd / cron 的静默 `rm`**，跟铁律 1 冲突。

---

## 8. 会话该长什么样

1. **摸现状** —— §1 五步，报 `df` + 各层大户
2. **分档给清单** —— A / B / C，含路径 / 大小 / 依据 / 代价；产物单独拎出来
3. **等用户点名** —— 「A 全清」「B1 不动」「不删 1h 内改过的」
4. **执行前复述** —— 展示准确的 `rm -rf` 和判定表（每项标「删 / 保留 + 理由 + 时间戳」）
5. **执行 + 逐项核对** —— 该删的删了 / 该留的还在 / 没点名的一个没动
6. **报 `sync` 后的 Avail 变化**，差额对不上要解释（比如同时有下载在跑）
7. **问要不要治根**（§7）
