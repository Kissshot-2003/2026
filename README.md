# 2026
Vibe coding记录

---

## 2026-10-21 · 把「Windows Terminal 里 ctrl+tab 没反应」挖到终端层

**起因**：[opencode#55606](https://github.com/anomalyco/opencode/issues/55606) 报告 `session.tab.next` 在 Windows Terminal 上静默失效。
issue 作者的判断是「Windows Terminal 把 ctrl+tab 绑给自己的 Terminal.NextTab，所以按键到不了应用」。

**先证伪自己的第一反应。** 第一反应是「TUI 没解码修饰键，补个解码器就行」——这恰恰是 OpenTUI 已经在做的事
（kitty CSI-u 早就支持了）。而 Windows Terminal 根本不实现 kitty 协议，所以那个方向是死的。

于是去读 Windows Terminal 的官方规范 #4999，确认了 `win32-input-mode`（DEC private mode 9001）这个格式：

```
CSI Vk ; Sc ; Uc ; Kd ; Cs ; Rc _
```

**然后在本机做实验，而不是靠推理。** 写了一个探针进程塞进 Windows Terminal 1.24，请求 `CSI ? 9001 h`，
再往里注入按键、把终端吐出来的每一个字节记录下来。三轮实验（legacy 模式 / win32 模式 / 加对照键）后拿到：

| 按键 | 应用实际收到的字节 |
|---|---|
| `ctrl+j`（WT 无默认绑定） | `Ctrl`↓ `J`↓ `J`↑ `Ctrl`↑ —— 完整按下+抬起 |
| `ctrl+shift+tab`（WT 绑定 Terminal.PrevTab） | `Shift`↓ `Ctrl`↓ `Tab`**↑（只有抬起）** `Shift`↑ `Ctrl`↑ |
| legacy 模式下的 `ctrl+shift+tab` | **一个字节都没有** |

扫描码（15=Tab、29=Ctrl、48=B、42=Shift）是真实值，这一下就把字段顺序钉死了：
`Vk;Sc;Uc;Kd;Cs;Rc`。`Cs` 是 Win32 的 `dwControlKeyState` 位掩码，实测 104/98/120 分别对应 Ctrl / Alt / Ctrl+Shift。

**结论比 issue 里说的更精确**：不是「Ctrl+Tab 跟 Tab 区分不开」，而是**终端把 key-down 吃掉了**，
连 win32 模式也只剩 key-up。所以任何输入解码都救不回来——用户必须去 `settings.json` 解绑。

顺带发现：OpenTUI 对 win32 序列完全不认识，`ESC[9;15;0;0;120;1_` 进来会变成 `name: ""` 的空按键，
**修饰键信息和连普通字符一起丢掉**。终端一旦处于 win32 模式，OpenTUI 就是全盲的。这是个独立的、真的 bug。

**三个产出**：

- [anomalyco/opentui#1726](https://github.com/anomalyco/opentui/pull/1726) — 新增 `parse.keypress-win32.ts` 解码器，
  24 个新测试（夹具全是实抓的序列 + 官方规范示例），1801 pass vs 1777 baseline，零回归。
- [anomalyco/opencode#57674](https://github.com/anomalyco/opencode/pull/57674) — 在 `cli/keybinds` 的会话标签表旁边
  补 Windows Terminal 小节，说明 `ctrl+tab` 被终端占用、给出 `alt+down`/`alt+up` 替代方案和 `unbound` 配置片段。
- [rothgar/awesome-tuis#957](https://github.com/rothgar/awesome-tuis/pull/957) — 补录 7 个漏收的 TUI。

**踩到的坑**：

- `SendKeys` / `SendInput` 往 WT 注入按键不稳定（焦点、修饰键顺序、结构体布局都有问题），
  最后靠**对照实验**（同一个终端、同一次运行里换一个没有绑定的键）才把结论做实——单看一个键的输出会得出错误结论。
- 在 Windows 上编辑文件，行尾符是 CRLF；`git` 提交进 GitHub 时归一化成 LF，
  于是用 API 传 blob 时得先转 LF，否则 diff 会显示整个文件被重写。
- PowerShell 5.1 按 ANSI 读 `.ps1` 文件，脚本里写中文路径会乱码；路径要当参数传进去。

## 2026-10-22 · 三种编码各测一遍，外加一个 OFD 解析崩溃

**补完上半场缺的那一格。** 上一次只测了「WT 绑着 ctrl+tab」和「开了 win32 模式」两种情况，
中间缺的是**解绑之后会怎样**。这次把实验做全了——改 `settings.json` 的 `actions` 数组解绑 ctrl+tab，
三种编码各跑一遍（改动前先做逐字节备份，跑完在 `finally` 里还原）：

| 编码 | `ctrl+tab` | `ctrl+shift+tab` | `tab` |
|---|---|---|---|
| 默认（WT 绑着） | 什么都没有 | 什么都没有 | `0x09` |
| 解绑 + legacy | `0x09` | `ESC [ Z` | `0x09` |
| 解绑 + 请求 kitty | `0x09` | `ESC [ Z` | `0x09` |
| 解绑 + win32 | `CSI 9 ; 15 ; 0 ; 1 ; 40 ; 1 _` | `CSI 9 ; 15 ; 0 ; 1 ; 56 ; 1 _` | `CSI 9 ; 1 ; 9 ; 1 ; 32 ; 1 _` |

三个结论：

1. **WT 完全忽略 kitty 协议**——请求 `CSI > 5 u` 的输出与 legacy 逐字节相同。
   所以 opencode 现在那个 kitty 请求在 Windows 上是个 no-op，修饰键状态对**所有**快捷键都不只是 ctrl+tab 不可用。
2. **解绑是必要但不充分**：legacy 编码下 `ctrl+tab` 就是 `0x09`（和纯 Tab 一样），
   `ctrl+shift+tab` 变成 `ESC[Z`（纯 shift+tab，Ctrl 丢了）。Ctrl 怎么都回不来。
3. **只有 win32 模式同时带来修饰键和 key-down**：`Cs = 0x28`（left ctrl）、`0x38`（left ctrl + shift），
   而且 `Kd = 1`——key-down 真的送到了。

**所以完整的修法是两半**：解绑终端的绑定 + 打开 win32-input-mode。后者要给 `@opentui/core` 加一个
native control kind（跟 kitty flags 同款），属于跨 Zig/ABI 的改动，建议由 opentui 维护者主导。

**顺手把文档 PR 的措辞改精确了。** 之前写「解绑 ctrl+tab 就能用」是错的——解绑只让按键**到达**应用，
legacy 编码下它仍然和纯 Tab 无法区分。文档现在说的是实际交付的字节。

**另一个 PR**：[renoyuan/easyofd#93](https://github.com/renoyuan/easyofd/pull/93) 修一个解析崩溃。
issue #90 里 `ofd:Clips/ofd:Path` 的 `@Boundary` 是可选的，但代码无条件 `float()` 它，
缺失时 `float("")` 抛 `ValueError`，**一个文本对象就能让整份 OFD 读不出来**。
改完补了 `test/test_clips_boundary.py`，修复前跑它正好复现那个 `ValueError`，修复后两个用例都过
（一个测缺失、一个测正常存在，防止改过头）。

**又踩了一个坑**：上游有个分支就叫 `fix`，所以 `fix/xxx` 这样的分支名推不上去——
git 报 `directory file conflict`（ref `fix` 是文件，`fix/xxx` 要的是目录）。换个不带斜杠前缀的名字就行。

**还有个坑**：GitHub 的 `matching-refs` 在这个仓库上会返回两千多个 ref，直接打印能把输出刷爆。
要查特定分支就用 `git/refs/heads/<name>` 单查。

## 2026-10-22 · 顺藤摸到中文文档的断层

**起因**：给 #57623 补评论时，发现那个 issue 说的「workflow 闸门写错了」**不是唯一的问题**。

`docs-update.yml` 和 `docs-locale-sync.yml` 同步的都是 `packages/web/src/content/docs/`，
而 `packages/web/astro.config.mjs` 里 `base: "/docs"`——这是 **v1 文档站**。
真正在长的 v2 文档在 `services/www/src/docs/content/`，`astro.config.ts` 里 `base: "/v2/"`。

两棵树不是一回事，语言环境也不一样：

| 树 | 部署路径 | 语言目录 |
|---|---|---|
| `packages/web/src/content/docs/` | `/docs` | 17 个（含 zh-cn、zh-tw） |
| `services/www/src/docs/content/` | `/v2/` | **一个都没有** |

所以就算把 `if: false` 修好，同步的仍然是 v1 的副本，v2 的页面还是没人翻。

**顺便量了断层有多大**（逐页对比标题数）：

| 页面 | 英文 | zh-cn |
|---|---|---|
| `providers.mdx` | 58 | 47 |
| `config.mdx` | 39 | 32 |
| `cli.mdx` | 29 | 25 |
| `troubleshooting.mdx` | 23 | 22 |
| `tools.mdx` | 19 | 18 |
| `permissions.mdx` | 11 | 10 |
| `keybinds.mdx` | 6 | 5 |
| `policies.mdx` | 5 | **没翻** |
| `references.mdx` | 6 | **没翻** |

33 个英文页面里 9 个落后，2 个压根没翻译。`keybinds.mdx` 缺的是整整一节 `## Binding Values`——
中文用户看文档根本不知道一个绑定值可以写多个快捷键。

**关键点**：同步被冻住**不影响**直接提交翻译。所以先补了最容易的那个：
[opencode#58017](https://github.com/anomalyco/opencode/pull/58017) 给 `zh-cn/keybinds.mdx`
补上 `## 绑定值` 一节。译名按 `.opencode/glossary/zh-cn.md` 和本页既有约定
（`keybind`/`shortcut` → 快捷键，`## Disable Keybind` 已译作 `## 禁用快捷键`，
所以 `## Binding Values` → `## 绑定值`），代码块和配置键名与英文版一字不改。

**验证的坑**：`services/www` 和 `packages/web` 是两棵独立的 Astro 树，
改 v1 的文档**要在 `packages/web` 里 build**，不能用 `services/www` 的 build 顶替——
两者都跑了一遍才算数。

**继续往里挖，逐页量出了具体缺什么。** 光看标题数会看错位置，得逐节对齐：

- `permissions.mdx` 缺的是**中间的** `## Auto mode`（自动模式）——不是末尾那节。
  缺一节会让后面整体错位一位，按下标看会以为缺的是最后一节。
- `troubleshooting.mdx` 缺 `## Uninstall`（卸载）。
- `tools.mdx` 的标题数对得上，但内容是**旧英文版的陈旧翻译**：工具已从 `patch`
  改名 `apply_patch`，中文还写着 `input.tool === "patch"`——照抄的钩子永远不会匹配。
  这是真正会坑到人的地方。

**写了个校验脚本**（`verify-translation.ts`），逐项比对中英两版：标题数与顺序、
代码块数量与 info string、内部链接数、表格行数，再加一条**每个锚点必须能在本页
解析到标题**。这里踩到两个自己的坑：标题正则把代码块里的 `# or` 当成了标题；
锚点比对也没考虑中文页面用中文标题做锚点（`#外部目录`）的站内惯例——脚本两处都修了。

**一个通用教训**：翻译任务里「结构对得上」不等于「内容对得上」。
`tools.mdx` 标题数完全一致，却是一份没人更新过的旧翻译。

## 2026-10-22 · 把校验脚本做成工具链，逐页把中文文档拉齐

**接着量 `cli.mdx` 和 `config.mdx`。** 之前的校验脚本只能给出总数（缺几个标题、
几个代码块、几行表格），定位不到具体位置。于是按「怎么对齐」重写了几支脚本：

| 脚本 | 干什么 |
|---|---|
| `verify-translation.ts` | 总数核对 + 锚点必须能在本页解析 |
| `diff-headings.ts` | 逐个标题并排，**围栏感知**（代码块里的 `# or` 不是标题） |
| `diff-blocks.ts` | 按**文件顺序**比表格/代码块的位置 |
| `diff-flags.ts` | 按**第一列的标志名**配对表格，列出缺的行 |
| `diff-code.ts` | 按代码块**首行的多重集**比对 |
| `diff-links.ts` | 内部链接集合差异 |
| `count-code-per-section.ts` | 每个小节的代码块数量 |
| `list-tables.ts` / `list-code.ts` | 列出全部表格/代码块 |

三个自己踩的坑，都写进脚本注释了：

- 表格行按 `|` 切开后，**第一列是 `r[1]` 不是 `r[0]`**（行首的 `|` 会多出一个空元素）。
  这个 off-by-one 让「所有表都匹配」变成了空集假象。
- **小节名是翻译过的**（「全局标志」≠ "Global Flags"），所以按小节名配对表全部落空，
  必须改成按标志名或按文件顺序。
- 贪心匹配会串位：中文表少一张时，后面的表会整体错位配对，报出一堆假的「缺失行」。

**结果**：`cli.mdx` 补上 `plugin`/`pr`/`db`/`debug` 四个命令小节、`auth login` 与
`export` 的标志表、`session delete` 子命令，以及 `--auto`/`--mdns`/`--mdns-domain`/`--cors`/`--pure`
几个后加的标志。`config.mdx` 补上托管设置、Shell、策略、图片附件、子代理深度、快照、
LSP 服务器七节。

**还发现两处是「内容错了」而不是「少了」**：

- `config.mdx` 的 TUI 一节中文版还在讲旧的 `tui` 选项放在 `opencode.json` 里，
  英文版已经挪到独立的 `tui.json` 文件了。
- `config.mdx` 的「主题」「快捷键」两处示例用的是 `opencode.json` + `config.json` schema，
  可它们本来就是 TUI 设置——**中文用户照抄会被 schema 拒绝**。

**又犯了一个错**：`git checkout -B <branch> origin/dev` 建分支时，
**工作区里未提交的改动被冲掉了**（permissions/troubleshooting/keybinds 等六个文件的成果），
而这些改动只存在于各自分支的提交里，只好重新 checkout 回来。
正确的做法是先 `git stash`。另外 `git checkout <branch> -- <file>` 会**暂存**文件，
所以跨分支切换时它们不会消失——这个特性救了场，但也导致有一次提交混进了 7 个文件。

**GitHub 依然拒绝这个 fork 的 PR 创建。** 把 fork 的 `dev` 同步到上游最新也没用。
分支、标题、正文、构建都备好了，只差「点提交」：`D:\通用项目\open-pending-prs.ps1`。


**分支管理上栽了个跟头**：建第二个、第三个翻译分支时**没有回到 `dev`**，于是它们逐级包含
前一个分支的提交——`zh-cn-references-policies` 实际含 3 个文件（重复了前一个 PR 的改动），
`zh-cn-missing-sections` 含 5 个。这既违反 PR 模板里「不含无关改动」的承诺，
标题和正文也对不上号。force-push 之后 GitHub 还把两个 PR 自动关了（diff 变空），
得重建。重建时改成**每个分支都从最新的 `origin/dev` 起**，只放自己那一个改动。

**还踩了两个 PowerShell 的坑**，都是同一种病：

- `git log --format=%B > msg.txt` 写出来的是 **UTF-16LE**，文件里每个字符后面都有 NUL，
  git 直接报 `a NUL byte in commit log message not allowed`。
  改用 `[System.IO.File]::WriteAllText($p, $text, (New-Object System.Text.UTF8Encoding($false)))`。
- 提交信息里带反引号时，PowerShell 的 here-string 会把它当转义符，整段解析失败。
  带反引号的提交信息一律先用文件写好，再 `git commit -F`。

**最后一个坑不是我的**：GitHub 突然拒绝了这个 fork 到上游的**所有** PR 创建——
连「已经有 PR 的分支」和「fork 自己的 dev 分支」都返回 404（而不是「PR 已存在」），
而 issue 创建、评论、其他仓库的 PR 都正常。限流配额是 4975/5000，状态页全绿。
看起来是 GitHub 对该 fork 的 PR 创建做了某种限流。改动已经推到 fork 上、构建也过了，
只差「点提交」这一步。




