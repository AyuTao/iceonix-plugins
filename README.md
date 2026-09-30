# 冰爪 Iceonix 插件

在 Claude Code、Codex 等 AI 工具里直接操作[冰爪](https://iceonix.com)：确认创作项目的正文和发布文案、扫码登录各平台、跟进发布进度。AI 负责写和改，真实发布由你 Mac 上的冰爪 App 在 Safari / 微信里执行。

> 本仓库由冰爪主仓库自动同步，请不要直接在这里提交修改。

## 使用前

1. 安装冰爪 App（macOS 15 及以上，Apple Silicon）：[iceonix.com](https://iceonix.com)。
2. 打开一次冰爪。App 启动时会把 MCP 桥接程序同步到 `~/Library/Application Support/Iceonix/HermesMCP/iceonix`，插件通过它和 App 通信。
3. 发布前在冰爪里完成平台登录和辅助功能授权。

插件只包含 MCP 配置和使用说明（Skill），不包含可执行文件，也不会收集账号密码。

## 安装

**Claude Code**

```bash
claude plugin marketplace add AyuTao/iceonix-plugins
claude plugin install iceonix@iceonix
```

**Codex**

```bash
codex plugin marketplace add AyuTao/iceonix-plugins
codex plugin add iceonix@iceonix
```

也可以在 Codex 里输入 `/plugins`，找到「冰爪 Iceonix」安装。

**Claude 桌面版、WorkBuddy**

在冰爪 App 的「设置 → Agent 连接」里点对应工具的「连接」，按提示重启即可。

## 怎么用

直接在对话里说，例如：

- 打开我最近的冰爪项目，我要确认正文和发布文案
- 看看我的冰爪各平台登录状态，需要时让我扫码
- 看看冰爪最近一批发布的进度
- 把这篇正文改写成小红书图文，保存到冰爪项目

冰爪面板出现的位置：

| 工具 | 面板位置 |
| --- | --- |
| Claude 桌面版 | 直接显示在对话里 |
| Codex | 右侧内置浏览器打开冰爪工作台 |
| Claude Code 等终端工具 | AI 给出本机链接，点开在浏览器里操作 |

在面板里可以改正文和图文草稿、扫码登录、确认后重试没成功的平台，底部的提示词按钮会把下一步发回对话。面板里的保存只改草稿；真实发布前 AI 会再向你确认一次。

## 更新

```bash
claude plugin marketplace update iceonix   # Claude Code
codex plugin marketplace upgrade           # Codex
```

插件版本与冰爪 App 同步，更新 App 后建议同时更新插件。

## 安全

- 一切都在你自己的 Mac 上运行：MCP 只连本机冰爪 App，工作台链接只监听 127.0.0.1。
- 账号密码、Cookie 与平台会话不经过 AI，也不经过插件。
- 真实发布（包括重发、重试）必须带上用户确认，冰爪会拒绝未经确认的请求。

## English

Iceonix plugin for Claude Code and Codex. It lets your AI assistant review Iceonix projects, handle platform QR logins and follow publishing progress, while the Iceonix macOS app performs the actual publishing on your Mac. Install the app from [iceonix.com](https://iceonix.com) and open it once, then:

```bash
claude plugin marketplace add AyuTao/iceonix-plugins && claude plugin install iceonix@iceonix
codex plugin marketplace add AyuTao/iceonix-plugins && codex plugin add iceonix@iceonix
```

## License

GPL-3.0-only
