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
